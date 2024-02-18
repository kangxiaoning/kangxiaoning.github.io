# Master重启导致集群异常

<show-structure depth="3"/>

一次变更引发集群异常，与预期不符，于是做了根因分析，记录下其中的关键技术原理。

## 1. 环境信息

### 1.1 组件版本

|     组件     |   版本    |
|:----------:|:-------:|
|   Linux    | 3.10.0  |
| Kubernetes | v1.18.6 |

### 1.2 集群架构

1. 集群有3个Master节点，所有Kubelet通过F5连接到Master。
2. Master-1和Master-2上分别运行了一个coreDNS，Master-3没有coreDNS。

![kubernetes arch](kubernetes-arch.svg)

## 2. 变更过程

如下变更在15分钟内完成，在重启Master-1和Master-2之前已确定coreDNS的Pod处于Ready状态，并且EndPoint完成更新。

| 步骤 |        变更操作        |
|:--:|:------------------:|
| 1  |     Master-3重启     |
| 2  | Master-1上驱逐coreDNS |
| 3  |     Master-1重启     |
| 4  | Master-2上驱逐coreDNS |
| 5  |     Master-2重启     |

## 3. 异常现象

1. 第4步执行后，应用出现DNS解析报错，大概30分钟后自行恢复。
2. 出现3个Kubelet NotReady，大量Kubelet报错“use of closed network connection”，大部分只报错1到3次后就恢复了，其中3个节点持续报错，直到重启Kubelet才恢复。

## 4. 分析过程

### 4.1 异常疑问

因为异常现象与预期不符，需要解释为什么会出现这些异常，我列了如下问题，根据这些问题指导分析方向。

1. 为什么第1个coreDNS驱逐后没报错？
2. 为什么第2个coreDNS驱逐后有报错？
3. 为什么30分钟能自行恢复？
4. 为什么Kubelet会NotReady？

### 4.2 异常复现

1. 采用同样的架构和组件版本，搭建一个全新的测试环境。
2. 在Master上利用iptables规则drop掉F5发来的数据包，达到模拟Master重启的场景。
3. 记录如下数据
   - 在Master，Node，F5上抓包，记录kube-proxy连接上的抓包数据。
   - 在Master，Node上执行netstat，观察TCP连接变化情况。

### 4.3 重要发现

在异常复现过程中，观察到如下现象。

1. Master异常后，**Node节点**上仍然能观察到`kube-proxy -> F5`这条TCP连接。
2. **`kube-proxy -> F5`正常收发包**，kube-proxy每30秒会向F5发送len为0的包，并且能收到F5的ACK回包。
3. `F5 -> Master`在Master异常大概3分30秒后，从**Master节点**上消失。
4. Master恢复后，在Node节点上`kube-proxy -> F5`依然存在，但是**F5不会向Master转发数据包**(观察到F5收到的包len都为0，不转发是F5的正常行为)。
4. 在Master异常大概30分钟后，kube-proxy收到一个**reset**包。
5. kube-proxy收到reset包后马上**释放了旧连接**，**建立了新连接**，至此kube-proxy与APIServer通信恢复正常。

### 4.4 根因分析

根据异常复现观察到的现象，尝试解释之前提出的疑问。

#### 4.4.1 为什么第1个coreDNS驱逐后没报错？

1. 第1个coreDNS驱逐后，集群中仍然有一个DNS Server可用，因此能正常提供DNS解析服务，没有报错符合预期。

 
#### 4.4.2 为什么第2个coreDNS驱逐后有报错？

1. Master重启导致kube-proxy与APIServer失去通信能力，kube-proxy无法从APIServer上Watch到Service/EndPoint变化，因此**无法更新Node上的iptables**规则。
2. coreDNS驱逐后IP发生了变化，原有的IP-A/IP-B，会变成IP-C/IP-A。
- 第1个coreDNS变更：IP-A -> IP-C
  - 变更中，IP-B提供DNS服务
  - 变更后，IP-B/IP-C提供DNS服务
  - 第一次变更过程及变更后，**IP-B始终提供服务**
- 第2个coreDNS变更：IP-B -> IP-A
  - 变更中，IP-C提供DNS服务，由于Node上的iptables规则没有更新，实际**没有可用的DNS Server**
  - 变更后，IP-C/IP-A供DNS服务，由于Node上的iptables规则没有更新，实际**可用的DNS Server只有IP-A**
3. 从上面分析得知，在第2个coreDNS变更过程中，会出现没有DNS服务的情况，因此一定会出现报错。

**新问题：** 第2个coreDNS变更后，很快IP-A就恢复了，即使Node上的iptables没有更新，按问题1的分析也可以使用IP-A解析，为什么没有恢复？

某些应用使用的镜像，DNS client会重用source port，这会导致请求始终命无效的conntrack记录，DNAT转换到不存在的coreDNS IP上，因此解析失败。之前确实观察到一次向不存在的IP发送解析请求，直到重启Pod才恢复。因为重启Pod会新建连接，就会新建conntrack，DNAT转换正确，解析恢复。

#### 4.4.3 为什么30分钟能自行恢复？

因为异常30分钟后，kube-proxy收到reset包，重建连接恢复了与APIServer的通信，此时触发了iptables更新，清理了失效的conntrack，所有网络连接恢复正常。reset包实际上是Master回的，F5只是转发了reset包给kube-proxy。

#### 4.4.4 为什么Kubelet会NotReady？

参考如下issue
- [](https://github.com/kubernetes/kubernetes/issues/87615)
- [](https://github.com/kubernetes/kubernetes/issues/87615#issuecomment-800031931)
- [](https://github.com/kubernetes/kubernetes/issues/87615#issuecomment-800828057)
- [](https://github.com/kubernetes/kubernetes/issues/87615#issuecomment-803517109)


#### 4.4.5 为什么kube-proxy没有检测到连接异常？

通过前面的分析已经可以解释30分钟恢复的原因，但是为什么kube-proxy没有检测到连接异常呢？经过分析是命中了如下bug。

- [](https://github.com/kubernetes/kubernetes/issues/107266)

#### 4.4.6 其它分析记录

1. 因为有3个Master，所以实际上每次重启Master只会影响最终连接到那台Master上的Node

## 5. 解决方案

1. 解决DNS解析异常

修改F5的**Action On Service Down**参数为**Reject**(默认是Node)，在检测到Master异常后及时通知kube-proxy，这样可以让kube-proxy重建连接，恢复与APIServer的通信。

- [](https://my.f5.com/manage/s/article/K15095)


2. 解决kubelet NotReady

根据[issue 87615](https://github.com/kubernetes/kubernetes/issues/87615)，需要升级或者重新编译kubelet。实际上我判断在我们环境中，修改F5参数也能解决，kubelet的做法是在client端提供异常检测及恢复功能，而修改F5参数相当于在Server端提供异常检测并通知client重建连接恢复。