# 集群升级分析

<show-structure depth="3"/>

Kubernetes升级分析记录。

网上大部分关于Kubernetes升级的文章，都没有考虑生产环境的业务影响，在参考时要谨慎判断。

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

| 公司      | 方案                                                                                                                                                                                                                                        |
|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 蚂蚁      | [蚂蚁大规模 Kubernetes 集群无损升级实践指南【探索篇】](https://segmentfault.com/a/1190000041374893)                                                                                                                                                           |
| vivo    | [Kubernetes 集群无损升级实践](https://segmentfault.com/a/1190000041145312)                                                                                                                                                                        |
| Google云 | [GKE 版本控制和支持](https://cloud.google.com/kubernetes-engine/versioning?hl=zh-cn)<br/>[标准集群升级](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-upgrades?hl=zh-cn)                                                                                                                                          |
| 阿里云     | [升级ACK集群](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/update-the-kubernetes-version-of-an-ack-cluster)<br/>[升级节点池](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/node-pool-updates) |

Google云的升级文档描述了升级过程需要[**移除节点上的Pod**](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-upgrades?hl=zh-cn#how-nodes-upgraded)，**未找到回退方案**。
> 节点升级方式
> 在节点池升级过程中，节点的升级方式取决于节点池升级策略以及配置方式。但是，基本步骤是一致的。为了升级节点，GKE 会**从节点中移除 Pod**，以便可以升级 Pod。
>
> 升级节点时，Pod 会发生以下情况：
> 
> 1. 该节点会被封锁，因此 Kubernetes 不会在其上安排新的 Pod。
> 2. 然后，节点会被排空，这意味着 Pod 会被移除。对于超额配置升级，GKE 遵循 Pod 的 PodDisruptionBudget 和 GracefulTerminationPeriod 设置，时间最长可达一小时。使用蓝绿升级时，如果您配置更长的过渡时间，则可以进行扩展。
> 3. 控制平面会将控制器管理的 Pod 重新调度到其他节点上。无法重新安排的 Pod 将保持待处理状态，直到可以重新安排为止。
> 
> 节点池升级过程可能需要几个小时，具体取决于升级策略、节点数量及其工作负载配置。

阿里云原地升级不影响业务，[**替盘升级要移除节点上的Pod**](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/node-pool-updates#600f2290460i4)，[**不支持回退**](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/node-pool-updates#p-19t-w8w-zdt)。

> 替盘升级单个节点内部的升级逻辑
> 1. 执行节点排水（并设置节点为不可调度）。
> 2. ECS关机，即停止节点。
> 3. 更换系统盘，系统盘ID会变更，但是云盘类型、实例IP地址以及弹性网卡MAC地址保持不变。
> 4. 重新初始化节点。
> 5. 重启节点，节点就绪（设置节点为可调度）。
> 
> [节点池升级后支持版本回退吗？](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/node-pool-updates#p-19t-w8w-zdt)
> 
> 目前，kubelet和容器运行时在版本升级后不支持版本回退。操作系统在升级后支持版本回退，前提只要原镜像仍旧为创建节点池所支持，就可以替换为原镜像。


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

## 3. 升级策略

升级这件事上，成本和风险要做取舍。

- 要**稳定**，考虑替换升级，风险可控，万一升级过程遇到预期外的异常，可以快速回退到原集群，确保服务正常。
- 要**成本**，考虑原地升级，风险较大，万一升级过程遇到预期外的异常，可能必须等到问题解决才能拉起服务，除非回退方案做的足够好。但是毕竟原来的运行环境已经变化了，回退方案也是一个新的变更，也有风险。

成本与应用强相关，如果应用能容忍按节点驱逐，即使替换迁移，成本也不会增加很多，比如旧集群下线的节点加入新集群使用；如果无法容忍按节点驱逐，意味着节点上的Pod迁移完成前，这个节点不能被新集群使用，因为无法按节点驱逐，导致清空一个节点的周期将会很长，具体取决于上面的应用什么时候迁移走。

## 4. 参考

记录下升级分析需要的一些信息。

- [Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [CVE-2021-25741](https://github.com/Betep0k/CVE-2021-25741)
- [Exploring Container Security: A Storage Vulnerability Deep Dive](https://security.googleblog.com/2021/12/exploring-container-security-storage.html)
  
