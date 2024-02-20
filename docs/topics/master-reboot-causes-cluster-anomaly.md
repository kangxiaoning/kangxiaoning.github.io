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

1. ~~第4步执行后~~，应用出现DNS解析报错，大概30分钟后自行恢复。

**update:** 得到应用反馈，**第2步执行后就出现DNS解析报错**，大概在第3步执行30分钟后自行恢复。

2. 出现3个Kubelet NotReady，大量Kubelet报错“use of closed network connection”，大部分只报错1到3次后就恢复了，其中3个节点持续报错，直到重启Kubelet才恢复。

## 4. 分析过程

### 4.1 异常疑问

因为异常现象与预期不符，需要解释为什么会出现这些异常，我列了如下问题，根据这些问题指导分析方向。

1. 为什么第1个coreDNS驱逐后没报错？

**update**: 后面重新查看应用日志，其实第1个coreDNS驱逐后已经产生DNS解析报错了。

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

1. 第1个coreDNS驱逐后，集群中仍然有一个DNS Server可用，因此能正常提供DNS解析服务，~~没有报错符合预期~~。

**update:** 后面重新查看应用日志，其实第1个coreDNS驱逐后已经产生DNS解析报错了，所以上述解释不合理。

技术原理上，第1个coreDNS驱逐后，连接Master-3的Nodes的iptables规则没有更新，所以这部分Nodes会有概率访问到不存在的coreDNS，一定会解析异常，但是由于集群中还有一个coreDNS可用，因此这时候重试可以恢复，报错较少，参考如下状态转换图，在驱逐第2个coreDNS后，集群中无效IP和Nodes最多，因此这个时候报错最多。
 
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

![iptable state transition](iptables-state-transition.svg)

3. 从上面分析得知，在第2个coreDNS变更过程中，会出现没有DNS服务的情况，因此一定会出现报错。

**新问题：** 第2个coreDNS变更后，很快IP-A就恢复了，即使Node上的iptables没有更新，按问题1的分析也可以使用IP-A解析，为什么没有恢复？

某些应用使用的镜像，DNS client会重用source port，这会导致请求始终命无效的conntrack记录，DNAT转换到不存在的coreDNS IP上，因此解析失败。之前确实观察到一次向不存在的IP发送解析请求，直到重启Pod才恢复。因为重启Pod会新建连接，就会新建conntrack，DNAT转换正确，解析恢复。

**update:** 参考4.4.1的解释，对照4.4.2的状态转换图，只要图中有标红的IP，这个阶段一定会出现解析异常，因此DNS解析异常时间段为第1台coreDNS删除到Master-1重启30分钟后，与观察到的应用现象一致。

#### 4.4.3 为什么30分钟能自行恢复？

因为异常30分钟后，kube-proxy收到reset包，重建连接恢复了与APIServer的通信，此时触发了iptables更新，清理了失效的conntrack，所有网络连接恢复正常。reset包实际上是Master回的，F5只是转发了reset包给kube-proxy。

#### 4.4.4 为什么Kubelet会NotReady？

参考如下issue
- [(1.17) Kubelet won't reconnect to Apiserver after NIC failure (use of closed network connection) #87615](https://github.com/kubernetes/kubernetes/issues/87615)
- [](https://github.com/kubernetes/kubernetes/issues/87615#issuecomment-800031931)
- [](https://github.com/kubernetes/kubernetes/issues/87615#issuecomment-800828057)
- [](https://github.com/kubernetes/kubernetes/issues/87615#issuecomment-803517109)


#### 4.4.5 为什么kube-proxy没有检测到连接异常？

通过前面的分析已经可以解释30分钟恢复的原因，但是为什么kube-proxy没有检测到连接异常呢？经过分析是命中了如下bug。

- [client wait forever if kube-apiserver restart in slb environment #107266](https://github.com/kubernetes/kubernetes/issues/107266)

#### 4.4.6 其它分析记录

1. 因为有3个Master，所以实际上每次重启Master只会影响最终连接到那台Master上的Node，参考4.4.2状态转换图。
2. 修改F5参数后，F5健康检查发现Master异常后，会主动向kube-proxy/kubelet发送reset包，重建连接后Node与Master通信恢复。

## 5. 解决方案

1. 解决DNS解析异常

修改F5的**Action On Service Down**参数为**Reject**(默认是Node)，在检测到Master异常后及时通知kube-proxy，这样可以让kube-proxy重建连接，恢复与APIServer的通信。F5每5秒检测一次，如果连续3次检测Member异常，1秒后将Member在Pool标记为Down，所以大概**在Master异常16秒后**会向Node发送reset包，此时通信恢复，kube-proxy更新iptables规则后解析正常。

- [K15095: Overview of the Action On Service Down feature](https://my.f5.com/manage/s/article/K15095)


2. 解决kubelet NotReady

根据[issue 87615](https://github.com/kubernetes/kubernetes/issues/87615)，需要升级或者重新编译kubelet。

修改F5参数能解决吗？

答：通过如下分析解决不了。

![use of closed network connection](closed-network-connection.svg)

上图描绘了`use of closed network connection`的场景。
1. Kubelet使用http库中的HTTP/2协议和Master通信。
2. Go的http库会维护一个connection pool，从pool里取出connection对象给kubelet使用。
3. Linux Kernel在协议栈维护真正的TCP Socket。
 
理论上connection pool里的connection对象和Kernel的socket一一对应，因为http库有bug，在维护connection pool的过程中会出现pool中有connection-A，Kernel中没有connection-A的情况。这时候kubelet使用connection-A就会出现`use of closed network connection`的报错，因为在Kernel中这个connection(socket)已经closed了。

从上图的通信路径来看，修复的地方有如下可能
1. 在图中标号为1的点修复，也就是**在kubelet中修复**，比如做端到端的探测，如果发现异常则重建连接。
2. 在图中标号为2的点修复，也就是**在Go的http库中修复**，确保pool中的connection都是有效的。

如果采用方案1，涉及到升级kubernetes版本，或者重新编译kubelet并替换这个组件，成本和风险都比较高。

如果采用方案2，在Go修复bug后，kubernetes项目要使用新的http库，改动会应用在在新版本，或者backport到旧版本等，同样涉及升级kubernetes。

Go和Kubernetes都是相对底层且应用很广泛的基础设施，这种级别的工程一旦有bug，影响都比较光且修复成本非常高。