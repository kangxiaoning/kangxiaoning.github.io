# 集群升级分析

<show-structure depth="3"/>

Kubernetes升级分析记录。

## 1. 社区策略

### 1.1 版本兼容

- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)

总结：一次只能升级一个minor版本，比如1.18.x升级到1.19.x。

### 1.2 升级方案

- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

- [How To Upgrade a Kubernetes Cluster | Upgrade ControlPlane & Worker Nodes in a Kubeadm Cluster](https://www.youtube.com/watch?v=tXlw8RauYM4)

## 2. 业界方案

### 2.1 原地升级

将kubernetes相关组件原地替换软件包，重启服务完成升级。

| 公司   | 方案                                                                              |
|------|---------------------------------------------------------------------------------|
| 蚂蚁   | [蚂蚁大规模 Kubernetes 集群无损升级实践指南【探索篇】](https://segmentfault.com/a/1190000041374893) |
| vivo | [Kubernetes 集群无损升级实践](https://segmentfault.com/a/1190000041145312)              |

原地升级要关注一些可能导致应用不可用的场景，升级过程是否能容忍？具体要结合自己环境的实际运维经验判断，比如下面一些Bug，和版本及架构等都有关系。

如下Bug可能导致Node出现NotReady，概率低，但是已经遇到过多次。
- [(1.17) Kubelet won't reconnect to Apiserver after NIC failure (use of closed network connection) #87615](https://github.com/kubernetes/kubernetes/issues/87615)

如下Bug在升级过程中可能引发coreDNS不可用，导致集群应用大面积异常。
- [client wait forever if kube-apiserver restart in slb environment #107266](https://github.com/kubernetes/kubernetes/issues/107266)

### 2.2 替换升级

新建一个高版本集群，将应用迁移应用到新集群，下线旧版本集群完成升级。

| 公司            | 方案                                                                                                  |
|---------------|-----------------------------------------------------------------------------------------------------|
| Alibaba Cloud | [How To Migrate Kubernetes Cluster With Zero Downtime](https://www.youtube.com/watch?v=wxh8Sv_WqEk) |

网上相关案例较少，猜测有如下原因。
1. 新为是新建集群，没有太多技术挑战。
2. 方案与应用强相关，不具备通用性。

根据应用运行环境，如下要考虑。
1. 防火墙，迁移可能会导致应用对外暴露的IP变化。
2. Chart，版本跨度大可能导致API变化，涉及Chart修改。
3. OS依赖，检查应用是否依赖OS，比如日志清理脚本，特殊内核参数等。
4. 成本，大集群的升级成本将会非常高。

## 3. 思考

升级这件事上，成本和风险要做取舍。

- 要**稳定**，考虑替换升级，风险可控，万一升级过程遇到预期外的异常，可以快速回退到原集群，确保服务正常。
- 要**成本**，考虑原地升级，风险较大，万一升级过程遇到预期外的异常，可能必须等到问题解决才能拉起服务，除非回退方案做的足够好。但是毕竟原来的运行环境已经变化了，回退方案也是一个新的变更，也有风险。

成本与应用强相关，如果应用能容忍按节点驱逐，即使替换迁移，成本也不会增加很多，比如旧集群下线的节点加入新集群使用；如果无法容忍按节点驱逐，意味着节点上的Pod迁移完成前，这个节点不能被新集群使用，因为无法按节点驱逐，导致清空一个节点的周期将会很长，具体取决于上面的应用什么时候迁移走。