# Time_squeeze和Netdev_budget

<show-structure depth="3"/>

在一次网络丢包案例分析中，深入到Linux内核及网卡驱动的代码实现才找到根因。因此这里总结下Linux内核网络栈中`time_squeeze`指标的含义、触发机制以及相关的性能优化方法。`time_squeeze`作为网络性能监控的重要指标，反映了内核在处理网络数据包时的效率和潜在瓶颈。通过对内核源码的详细分析，特别是Mellanox网卡驱动的实现细节，展示了从硬件中断到软中断处理的完整数据包处理流程，结合实际案例给出了性能调优建议。

> 本文参考代码版本为：[openeuler 4.19.90-2404.2.0](https://gitee.com/openeuler/kernel/tree/4.19.90-2404.2.0/)，不同kernel版本可能会有差异。
> 
{style="note"}

time_squeeze持续增长，说明`softirq`的收包预算用完时`TX/RX queue`中仍然有包等待处理。

只要**TX/RX queue**在下次`softirq`处理之前没有**overflow**，那就不会因为`time_squeeze`增加而导致丢包，但如果`time_squeeze`持续增加，就有`TX/RX queue`溢出导致丢包的可能，需要结合**packet_drop**指标来看。

## 1. /proc/net/softnet_stat

`/proc/net/softnet_stat`文件提供了每个CPU核心的网络软中断处理统计信息。该文件中的每一行代表一个CPU核心的网络处理统计数据，每列数据都反映了网络栈不同层面的处理情况，通过如下代码可知对应列的含义。

- Column-01: **packet_process**: the number of frames received by the interrupt handler.
- Column-02: **packet_drop**: the number of frames dropped due to netdev_max_backlog being exceeded.
- Column-03: **time_squeeze**: the number of times ksoftirqd ran out of netdev_budget or CPU time when there was still work to be done.
- Column-09: **cpu_collision**: collision occur while obtaining device lock while transmitting.
- Column-10: **received_rps**: number of times cpu woken up received_rps.
- Column-11: **flow_limit_count**: number of times reached flow limit count.


`softnet_seq_show`函数负责将内核中的`softnet_data`结构体数据格式化输出到`/proc/net/softnet_stat`文件。该函数通过`seq_printf`将各项统计数据以十六进制格式输出，方便用户空间程序读取和分析网络性能数据。其中的`flow_limit_count`需要在启用`CONFIG_NET_FLOW_LIMIT`配置时才会有实际数值。

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/net/core/net-procfs.c

static int softnet_seq_show(struct seq_file *seq, void *v)
{
	struct softnet_data *sd = v;
	unsigned int flow_limit_count = 0;

#ifdef CONFIG_NET_FLOW_LIMIT
	struct sd_flow_limit *fl;

	rcu_read_lock();
	fl = rcu_dereference(sd->flow_limit);
	if (fl)
		flow_limit_count = fl->count;
	rcu_read_unlock();
#endif

	seq_printf(seq,
		   "%08x %08x %08x %08x %08x %08x %08x %08x %08x %08x %08x\n",
		   sd->processed, sd->dropped, sd->time_squeeze, 0,
		   0, 0, 0, 0, /* was fastroute */
		   0,	/* was cpu_collision */
		   sd->received_rps, flow_limit_count);
	return 0;
}
```
{collapsible="true" collapsed-title="softnet_seq_show" default-state="collapsed"}

- Each line of `softnet_stat` represents the packets a CPU core has received.
- Upon traffic coming into the NIC, a hard interrupt is generated. An IRQ handler acknowledges the hard interrupt.
- A SoftIRQ is scheduled to pull packets off the NIC at a time when it will not interrupt another process.
- The SoftIRQ runs to pull `net.core.netdev_budget` packets off the NIC
- Normally, packets are processed through network protocols as the packets are received.
- Packets may be stored in a backlog queue of approximate size `net.core.netdev_max_backlog` under certain conditions - such as when RPS is configured or the receiving device is virtual. If the `backlog` is full, the second column of `softnet_stat` will increment.
- If there were packets left when `budget` was over, the third column of `softnet_stat` will increment
- You can view `softnet_stat` in realtime whilst reproducing packet loss to determine if `backlog` and/or `budget` should be incremented
- You can change the `netdev` tunables with `sysctl -w` in realtime

参考：**[How to tune `net.core.netdev_max_backlog` and `net.core.netdev_budget` sysctl kernel tunables?](https://access.redhat.com/solutions/1241943)**

## 2. time_squeeze

### 2.1 time_squeeze含义

If the SoftIRQs do not run for long enough, the rate of incoming data could exceed the kernel's capability to drain the buffer fast enough. As a result, the NIC buffers will overflow and traffic will be lost. Occasionally, it is necessary to increase the time that SoftIRQs are allowed to run on the CPU. This is known as the netdev_budget. The default value of the budget is 300. This will cause the SoftIRQ process to drain 300 messages from the NIC before getting off the CPU:

```Shell
# sysctl net.core.netdev_budget 
net.core.netdev_budget = 300 
```

参考： **[Red Hat Enterprise Linux Network Performance Tuning Guide](https://access.redhat.com/articles/1391433)**

### 2.2 time_squeeze触发代码

`net_rx_action`函数是`NET_RX_SOFTIRQ`软中断的处理函数，负责处理网络接收的核心逻辑。该函数实现了NAPI（New API）机制，通过轮询的方式批量处理网络数据包，提高了网络处理效率。函数使用两个关键参数控制处理时长：`netdev_budget`（处理数据包数量预算）和`netdev_budget_usecs`（处理时间预算）。当任一预算耗尽时，函数会停止处理并更新`time_squeeze`计数器。

通过全局搜索可以发现**time_squeeze**只在`net_rx_action()`这个函数被修改，而这个函数是`NET_RX_SOFTIRQ`中断的处理函数，所以`time_squeeze`的变化可以反映内核处理`NET_RX_SOFTIRQ`中断的效率。

如果在处理过程中预算耗尽或者超时，函数会增加time_squeeze计数器，并退出循环。

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/net/core/dev.c

static __latent_entropy void net_rx_action(struct softirq_action *h)
{
	struct softnet_data *sd = this_cpu_ptr(&softnet_data);
	unsigned long time_limit = jiffies +
		usecs_to_jiffies(READ_ONCE(netdev_budget_usecs));
	int budget = READ_ONCE(netdev_budget);
	LIST_HEAD(list);
	LIST_HEAD(repoll);

	local_irq_disable();
	list_splice_init(&sd->poll_list, &list);
	local_irq_enable();

	for (;;) {
		struct napi_struct *n;

		if (list_empty(&list)) {
			if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
				goto out;
			break;
		}

		n = list_first_entry(&list, struct napi_struct, poll_list);
		budget -= napi_poll(n, &repoll);

		/* If softirq window is exhausted then punt.
		 * Allow this to run for 2 jiffies since which will allow
		 * an average latency of 1.5/HZ.
		 */
		if (unlikely(budget <= 0 ||
			     time_after_eq(jiffies, time_limit))) {
			sd->time_squeeze++;
			break;
		}
	}

	local_irq_disable();

	list_splice_tail_init(&sd->poll_list, &list);
	list_splice_tail(&repoll, &list);
	list_splice(&list, &sd->poll_list);
	if (!list_empty(&sd->poll_list))
		__raise_softirq_irqoff(NET_RX_SOFTIRQ);

	net_rps_action_and_irq_enable(sd);
out:
	__kfree_skb_flush();
}
```
{collapsible="true" collapsed-title="net_rx_action" default-state="collapsed"}

## 3. Mellanox驱动处理逻辑

这里主要分析`net_rx_action()`中`budget -= napi_poll(n, &repoll);`这一行涉及到的代码，也就是网卡驱动注册到内核的`poll()`方法的逻辑。

### 3.1 网卡驱动的`poll()`方法

Mellanox网卡驱动通过`net_device_ops`结构体向内核注册各种网络操作函数。当网卡启动时（通过`ifconfig up`或`ip link set up`），内核会调用`ndo_open`函数，在Mellanox驱动中对应`mlx5e_open`函数。该函数会完成网卡的初始化工作，包括分配资源、创建队列、注册中断处理函数等。其中最重要的是注册NAPI poll函数，这是实现高效网络数据包处理的关键。

根据网卡驱动定义的`net_device_ops`可知，在Mellanox中`ndo_open`的实现是`mlx5e_open`，up网卡时会调用这个函数，最终会注册`poll()`方法。

```C
const struct net_device_ops mlx5e_netdev_ops = {
	.ndo_open                = mlx5e_open,
	.ndo_stop                = mlx5e_close,
	.ndo_start_xmit          = mlx5e_xmit,
	.ndo_setup_tc            = mlx5e_setup_tc,
	.ndo_select_queue        = mlx5e_select_queue,
	.ndo_get_stats64         = mlx5e_get_stats,
	.ndo_set_rx_mode         = mlx5e_set_rx_mode,
	.ndo_set_mac_address     = mlx5e_set_mac,
	.ndo_vlan_rx_add_vid     = mlx5e_vlan_rx_add_vid,
	.ndo_vlan_rx_kill_vid    = mlx5e_vlan_rx_kill_vid,
	.ndo_set_features        = mlx5e_set_features,
	.ndo_fix_features        = mlx5e_fix_features,
	.ndo_change_mtu          = mlx5e_change_nic_mtu,
	.ndo_do_ioctl            = mlx5e_ioctl,
	.ndo_set_tx_maxrate      = mlx5e_set_tx_maxrate,
	.ndo_udp_tunnel_add      = mlx5e_add_vxlan_port,
	.ndo_udp_tunnel_del      = mlx5e_del_vxlan_port,
	.ndo_features_check      = mlx5e_features_check,
	.ndo_tx_timeout          = mlx5e_tx_timeout,
	.ndo_bpf		 = mlx5e_xdp,
	.ndo_xdp_xmit            = mlx5e_xdp_xmit,
#ifdef CONFIG_MLX5_EN_ARFS
	.ndo_rx_flow_steer	 = mlx5e_rx_flow_steer,
#endif
#ifdef CONFIG_MLX5_ESWITCH
	/* SRIOV E-Switch NDOs */
	.ndo_set_vf_mac          = mlx5e_set_vf_mac,
	.ndo_set_vf_vlan         = mlx5e_set_vf_vlan,
	.ndo_set_vf_spoofchk     = mlx5e_set_vf_spoofchk,
	.ndo_set_vf_trust        = mlx5e_set_vf_trust,
	.ndo_set_vf_rate         = mlx5e_set_vf_rate,
	.ndo_get_vf_config       = mlx5e_get_vf_config,
	.ndo_set_vf_link_state   = mlx5e_set_vf_link_state,
	.ndo_get_vf_stats        = mlx5e_get_vf_stats,
	.ndo_has_offload_stats	 = mlx5e_has_offload_stats,
	.ndo_get_offload_stats	 = mlx5e_get_offload_stats,
#endif
};
```
{collapsible="true" collapsed-title="mlx5e_netdev_ops" default-state="collapsed"}

注册`poll()`的调用过程如下。

```plantuml
@startuml
:mlx5e_open();
:mlx5e_open_locked();
:mlx5e_open_channels();
:mlx5e_open_channel();
split
 :netif_napi_add();
  note right
   注册`mlx5e_napi_poll()`;
  end note
split again
 :napi_enable();
end split
@enduml
```

注：更简单的方法是在在网卡驱动代码中搜索`netif_napi_add`关键字，即可找到`poll()`方法的具体实现，如下是Mellanox网卡驱动的搜索结果。

```C
	netif_napi_add(netdev, &c->napi, mlx5e_napi_poll, 64);
```

`netif_napi_add`函数是NAPI机制的核心注册函数，它将网卡驱动的poll函数注册到内核的NAPI框架中。该函数初始化NAPI结构体，设置poll回调函数和权重值（weight），并将NAPI实例添加到网络设备的NAPI列表中。权重值决定了每次poll调用最多可以处理的数据包数量，这是防止单个网卡垄断CPU资源的重要机制。函数还会初始化GRO（Generic Receive Offload）哈希表，用于后续的数据包聚合处理。

`netif_napi_add()`函数定义如下，查看函数实现了解每个输入参数的含义。
```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/net/core/dev.c

void netif_napi_add(struct net_device *dev, struct napi_struct *napi,
		    int (*poll)(struct napi_struct *, int), int weight)
{
	INIT_LIST_HEAD(&napi->poll_list);
	hrtimer_init(&napi->timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_PINNED);
	napi->timer.function = napi_watchdog;
	init_gro_hash(napi);
	napi->skb = NULL;
	napi->poll = poll;
	if (weight > NAPI_POLL_WEIGHT)
		pr_err_once("netif_napi_add() called with weight %d on device %s\n",
			    weight, dev->name);
	napi->weight = weight;
	napi->dev = dev;
#ifdef CONFIG_NETPOLL
	napi->poll_owner = -1;
#endif
	set_bit(NAPI_STATE_SCHED, &napi->state);
	set_bit(NAPI_STATE_NPSVC, &napi->state);
	list_add_rcu(&napi->dev_list, &dev->napi_list);
	napi_hash_add(napi);
}
EXPORT_SYMBOL(netif_napi_add);
```
{collapsible="true" collapsed-title="netif_napi_add(struct net_device *dev, struct napi_struct *napi, int (*poll)(struct napi_struct *, int), int weight)" default-state="collapsed"}

### 3.2 mlx5e_napi_poll()

`mlx5e_napi_poll`是Mellanox网卡驱动的NAPI poll处理函数，它在软中断上下文中执行，负责批量处理网络数据包。该函数采用了发送优先的策略，首先清理发送队列中已完成的数据包，释放相关资源，然后再处理接收队列中的新数据包。这种设计可以及时释放发送缓冲区，避免发送队列满导致的网络拥塞。函数还实现了动态中断调节（DIM），根据网络负载自动调整中断聚合参数，在低延迟和高吞吐量之间取得平衡。

在`net_rx_action()`函数中，`budget -= napi_poll(n, &repoll);`最终会执行到网卡驱动的`poll()`方法，上面分析可知具体是`mlx5e_napi_poll()`方法。

通过下面代码及注释，可以看到主要执行了如下操作。
1. 清理发送队列的ring buffer
2. 处理接收队列的数据包

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/drivers/net/ethernet/mellanox/mlx5/core/en_txrx.c

int mlx5e_napi_poll(struct napi_struct *napi, int budget)
{
	struct mlx5e_channel *c = container_of(napi, struct mlx5e_channel,
					       napi);
	struct mlx5e_ch_stats *ch_stats = c->stats;
	bool busy = false;
	int work_done = 0;
	int i;

	ch_stats->poll++;

	for (i = 0; i < c->num_tc; i++)
		busy |= mlx5e_poll_tx_cq(&c->sq[i].cq, budget); // 清理发送队列的ring buffer

	busy |= mlx5e_poll_xdpsq_cq(&c->xdpsq.cq);

	if (c->xdp)
		busy |= mlx5e_poll_xdpsq_cq(&c->rq.xdpsq.cq);

	if (likely(budget)) { /* budget=0 means: don't poll rx rings */
		work_done = mlx5e_poll_rx_cq(&c->rq.cq, budget); // 处理接收队列的数据包
		busy |= work_done == budget;
	}

	busy |= c->rq.post_wqes(&c->rq);

	if (busy) {
		if (likely(mlx5e_channel_no_affinity_change(c)))
			return budget;
		ch_stats->aff_change++;
		if (budget && work_done == budget)
			work_done--;
	}

	if (unlikely(!napi_complete_done(napi, work_done)))
		return work_done;

	ch_stats->arm++;

	for (i = 0; i < c->num_tc; i++) {
		mlx5e_handle_tx_dim(&c->sq[i]);
		mlx5e_cq_arm(&c->sq[i].cq);
	}

	mlx5e_handle_rx_dim(&c->rq);

	mlx5e_cq_arm(&c->rq.cq);
	mlx5e_cq_arm(&c->icosq.cq);
	mlx5e_cq_arm(&c->xdpsq.cq);

	return work_done;
}
```
{collapsible="true" collapsed-title="mlx5e_napi_poll" default-state="collapsed"}


### 3.3 mlx5e_poll_tx_cq()

`mlx5e_poll_tx_cq`函数负责处理发送完成队列（TX CQ），清理已经成功发送的数据包。该函数通过遍历完成队列中的完成事件（CQE），释放相关的DMA映射和socket buffer。函数使用了独立的TX处理预算（MLX5E_TX_CQ_POLL_BUDGET=128），这个预算独立于NAPI的总预算，确保发送清理不会过度占用CPU资源。当检测到硬件错误时，函数会触发错误恢复机制。处理完成后，如果发送队列之前因为满而停止，函数会检查是否有足够空间并唤醒队列。

`mlx5e_poll_tx_cq()`的主要作用是清理ring buffer，在循环中持续运行，直到满足如下条件才退出执行。
1. 超过MLX5E_TX_CQ_POLL_BUDGET预算，默认是128
2. 或者没有更多的CQE(Completion Queue Entry)可用

**注意**：有可能一次`mlx5e_poll_tx_cq()`运行超出预算，但是ring buffer未清理完。如果发送包的速率大于清理ring buffer的速率，可能出现ring buffer溢出。

```C
#define MLX5E_TX_CQ_POLL_BUDGET        128
```

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/drivers/net/ethernet/mellanox/mlx5/core/en_tx.c

bool mlx5e_poll_tx_cq(struct mlx5e_cq *cq, int napi_budget)
{
    struct mlx5e_sq_stats *stats; // 发送队列统计信息
    struct mlx5e_txqsq *sq;       // 发送队列结构体
    struct mlx5_cqe64 *cqe;       // 完成队列事件
    u32 dma_fifo_cc;              // DMA FIFO计数器
    u32 nbytes;                   // 处理的总字节数
    u16 npkts;                    // 处理的数据包数量
    u16 sqcc;                     // 发送队列计数器
    int i;                        // 循环计数

    // 获取发送队列结构体
    sq = container_of(cq, struct mlx5e_txqsq, cq);

    // 确保发送队列处于启用状态
    if (unlikely(!test_bit(MLX5E_SQ_STATE_ENABLED, &sq->state)))
        return false;

    // 获取完成队列中的完成事件
    cqe = mlx5_cqwq_get_cqe(&cq->wq);
    if (!cqe)
        return false;

    // 初始化统计信息
    stats = sq->stats;

    npkts = 0;
    nbytes = 0;

    // 更新发送队列计数器
    sqcc = sq->cc;

    // 避免频繁更新缓存线
    dma_fifo_cc = sq->dma_fifo_cc;

    // 循环处理完成事件
    i = 0;
    do {
        u16 wqe_counter;           // 工作队列元素计数
        bool last_wqe;             // 是否为最后一个WQE

        // 移除完成事件
        mlx5_cqwq_pop(&cq->wq);

        // 获取工作队列元素计数
        wqe_counter = be16_to_cpu(cqe->wqe_counter);

        // 处理错误事件
        if (unlikely(cqe->op_own >> 4 == MLX5_CQE_REQ_ERR)) {
            if (!test_and_set_bit(MLX5E_SQ_STATE_RECOVERING, &sq->state)) {
                // 处理错误事件并安排恢复工作
                mlx5e_dump_error_cqe(sq, (struct mlx5_err_cqe *)cqe);
                queue_work(cq->channel->priv->wq, &sq->recover.recover_work);
            }
            stats->cqe_err++;      // 统计错误事件数量
        }

        // 遍历WQE中的所有DMA映射
        do {
            struct mlx5e_tx_wqe_info *wi; // WQE信息
            struct sk_buff *skb;          // 数据包
            u16 ci;                       // WQE索引
            int j;                        // 内部循环计数

            // 获取最后一个WQE标志
            last_wqe = (sqcc == wqe_counter);

            // 获取WQE索引
            ci = mlx5_wq_cyc_ctr2ix(&sq->wq, sqcc);
            wi = &sq->db.wqe_info[ci];
            skb = wi->skb;

            // 如果是NOP，则跳过
            if (unlikely(!skb)) {
                sqcc++;
                continue;
            }

            // 如果设置了硬件时间戳，则更新数据包的时间戳
            if (unlikely(skb_shinfo(skb)->tx_flags & SKBTX_HW_TSTAMP)) {
                struct skb_shared_hwtstamps hwts = {};
                hwts.hwtstamp = mlx5_timecounter_cyc2time(sq->clock, get_cqe_ts(cqe));
                skb_tstamp_tx(skb, &hwts);
            }

            // 解映射DMA缓冲区，也就是清理ring buffer
            for (j = 0; j < wi->num_dma; j++) {
                struct mlx5e_sq_dma *dma = mlx5e_dma_get(sq, dma_fifo_cc++);
                mlx5e_tx_dma_unmap(sq->pdev, dma);
            }

            // 更新统计信息
            npkts++;
            nbytes += wi->num_bytes;
            sqcc += wi->num_wqebbs;
            napi_consume_skb(skb, napi_budget);
        } while (!last_wqe); // 直到最后一个wqe完成才结束循环
    } while ((++i < MLX5E_TX_CQ_POLL_BUDGET) && (cqe = mlx5_cqwq_get_cqe(&cq->wq))); // 超过MLX5E_TX_CQ_POLL_BUDGET预算，或者没有更多的cqe则结束循环

    // 更新处理的事件数量
    stats->cqes += i;

    // 更新数据库记录以释放完成队列的空间
    mlx5_cqwq_update_db_record(&cq->wq);

    // 确保完成队列空间被释放
    wmb();

    // 更新DMA FIFO计数器和发送队列计数器
    sq->dma_fifo_cc = dma_fifo_cc;
    sq->cc = sqcc;

    // 唤醒发送队列
    netdev_tx_completed_queue(sq->txq, npkts, nbytes);

    // 如果发送队列被停止且有足够的空间，则唤醒队列
    if (netif_tx_queue_stopped(sq->txq) &&
        mlx5e_wqc_has_room_for(&sq->wq, sq->cc, sq->pc, MLX5E_SQ_STOP_ROOM) &&
        !test_bit(MLX5E_SQ_STATE_RECOVERING, &sq->state)) {
        netif_tx_wake_queue(sq->txq);
        stats->wake++;
    }

    // 返回是否达到处理预算
    return (i == MLX5E_TX_CQ_POLL_BUDGET);
}
```

### 3.4 mlx5e_poll_rx_cq()

`mlx5e_poll_rx_cq`函数是接收路径的核心处理函数，负责从接收完成队列中提取数据包并交给上层协议栈处理。该函数支持压缩CQE（Compressed CQE）特性，可以在单个CQE中包含多个数据包的信息，提高了PCIe带宽利用率。函数通过预算机制控制每次处理的数据包数量，确保系统的公平性和响应性。处理过程中还集成了XDP（eXpress Data Path）支持，可以在驱动层直接处理某些数据包，大幅提升特定场景下的性能。

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/drivers/net/ethernet/mellanox/mlx5/core/en_rx.c

int mlx5e_poll_rx_cq(struct mlx5e_cq *cq, int budget)
{
    struct mlx5e_rq *rq = container_of(cq, struct mlx5e_rq, cq); // 获取与CQ关联的接收队列
    struct mlx5e_xdpsq *xdpsq = &rq->xdpsq;
    struct mlx5_cqe64 *cqe; // 定义CQE指针
    int work_done = 0; // 初始化已处理的工作量

    // 如果接收队列未启用，则直接返回0
    if (unlikely(!test_bit(MLX5E_RQ_STATE_ENABLED, &rq->state)))
        return 0;

    // 如果存在待解压缩的CQ条目
    if (cq->decmprs_left) {
        // 处理解压缩的CQ条目，并更新已处理的工作量
        work_done += mlx5e_decompress_cqes_cont(rq, cq, 0, budget);
        // 如果还有待解压缩的CQ条目或已达到预算限制，则结束处理
        if (cq->decmprs_left || work_done >= budget)
            goto out;
    }

    // 获取CQ中的下一个CQE
    cqe = mlx5_cqwq_get_cqe(&cq->wq);
    // 如果没有CQE，则根据已处理的工作量情况决定是否返回
    if (!cqe) {
        if (unlikely(work_done))
            goto out;
        return 0;
    }

    // 循环处理CQ条目
    do {
        // 如果CQE是压缩格式，则开始解压缩并更新已处理的工作量
        if (mlx5_get_cqe_format(cqe) == MLX5_COMPRESSED) {
            work_done += mlx5e_decompress_cqes_start(rq, cq, budget - work_done);
            continue;
        }

        // 从CQ中移除当前CQE
        mlx5_cqwq_pop(&cq->wq);

        // 调用处理函数处理当前CQE，最终调用的是`mlx5e_handle_rx_cqe()`
        rq->handle_rx_cqe(rq, cqe);
    } while ((++work_done < budget) && (cqe = mlx5_cqwq_get_cqe(&cq->wq))); // 有预算并且有CQE的情况下循环处理

out:
    if (xdpsq->doorbell) {
        mlx5e_xmit_xdp_doorbell(xdpsq);
        xdpsq->doorbell = false;
    }

    if (xdpsq->redirect_flush) {
        xdp_do_flush_map();
        xdpsq->redirect_flush = false;
    }

    mlx5_cqwq_update_db_record(&cq->wq);

    // 确保在释放CQ空间之前更新已完成的工作量
    wmb();

    // 返回已处理的工作量
    return work_done;
}
```

接收处理函数的初始化过程涉及多个层次的函数指针设置。在创建接收队列时，驱动会根据配置选择合适的接收处理函数。对于标准的单包接收模式，使用`mlx5e_handle_rx_cqe`函数；对于多包接收模式（Multi-Packet WQE），则使用`mlx5e_handle_rx_cqe_mpwrq`函数。这种设计允许驱动在运行时根据不同的硬件特性和配置选择最优的处理路径。

在`mlx5e_alloc_rq()`初始化了`rq->handle_rx_cqe`。

```C
			rq->handle_rx_cqe = c->priv->profile->rx_handlers.handle_rx_cqe;
```

根据`mlx5e_profile`初始化可知，`rq->handle_rx_cqe`最终被初始化为`mlx5e_handle_rx_cqe`。

```C
static const struct mlx5e_profile mlx5e_nic_profile = {
	.init		   = mlx5e_nic_init,
	.cleanup	   = mlx5e_nic_cleanup,
	.init_rx	   = mlx5e_init_nic_rx,
	.cleanup_rx	   = mlx5e_cleanup_nic_rx,
	.init_tx	   = mlx5e_init_nic_tx,
	.cleanup_tx	   = mlx5e_cleanup_nic_tx,
	.enable		   = mlx5e_nic_enable,
	.disable	   = mlx5e_nic_disable,
	.update_stats	   = mlx5e_update_ndo_stats,
	.max_nch	   = mlx5e_get_max_num_channels,
	.update_carrier	   = mlx5e_update_carrier,
	.rx_handlers.handle_rx_cqe       = mlx5e_handle_rx_cqe,
	.rx_handlers.handle_rx_cqe_mpwqe = mlx5e_handle_rx_cqe_mpwrq,
	.max_tc		   = MLX5E_MAX_NUM_TC,
};
```

### 3.5 mlx5e_handle_rx_cqe()

`mlx5e_handle_rx_cqe`函数是数据包接收的最后一步，负责将硬件接收的数据构建成内核可以处理的socket buffer（skb）。该函数首先从CQE中提取数据包信息，包括数据长度、校验和状态等，然后调用相应的函数重构skb。如果启用了XDP，函数会先尝试XDP处理路径。构建好的skb会通过`napi_gro_receive`提交给网络栈，利用GRO机制聚合小包，提高协议栈处理效率。处理完成后，函数会释放使用过的接收缓冲区，为后续接收准备空间。

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/drivers/net/ethernet/mellanox/mlx5/core/en_rx.c

void mlx5e_handle_rx_cqe(struct mlx5e_rq *rq, struct mlx5_cqe64 *cqe)
{
    struct mlx5_wq_cyc *wq = &rq->wqe.wq; // 获取请求队列中的循环工作队列结构
    struct mlx5e_wqe_frag_info *wi;        // 定义工作队列元素片段信息指针
    struct sk_buff *skb;                   // 定义网络缓冲区结构指针
    u32 cqe_bcnt;                          // 定义CQE字节计数值
    u16 ci;                                // 定义工作队列元素计数器索引

    // 从CQE中获取工作队列元素计数器索引
    ci       = mlx5_wq_cyc_ctr2ix(wq, be16_to_cpu(cqe->wqe_counter));
    wi       = get_frag(rq, ci);           // 根据索引获取对应的工作队列元素片段信息
    cqe_bcnt = be32_to_cpu(cqe->byte_cnt); // 从CQE中获取字节计数值

    // 从工作队列元素中重构网络缓冲区skb
    skb = rq->wqe.skb_from_cqe(rq, cqe, wi, cqe_bcnt);
    if (!skb) {                            // 如果skb为空
        /* 处理可能的XDP情况 */
        if (__test_and_clear_bit(MLX5E_RQ_FLAG_XDP_XMIT, rq->flags)) {
            /* 不释放页面，等待XDP_TX完成后释放 */
            goto wq_cyc_pop;
        }
        goto free_wqe;                     // 直接跳转到释放WQE
    }

    // 完成接收CQE的处理，并准备接收skb
    mlx5e_complete_rx_cqe(rq, cqe, cqe_bcnt, skb);
    napi_gro_receive(rq->cq.napi, skb);    // 注册接收skb，后续就一步一步送到协议栈了

free_wqe:                                 // 释放接收WQE
    mlx5e_free_rx_wqe(rq, wi, true);

wq_cyc_pop:                               // 从工作队列中移除已处理的元素
    mlx5_wq_cyc_pop(wq);
}
```

参考：[](https://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)

## 4. 性能参数优化

当系统出现`time_squeeze`持续增长时，表明软中断处理存在性能瓶颈。这通常意味着在分配的时间或预算内，内核无法处理完所有待处理的网络数据包。解决这个问题的关键是增加软中断的处理能力，主要通过调整`netdev_budget`和`netdev_budget_usecs`两个参数来实现。这两个参数分别控制了每次软中断可以处理的最大数据包数量和最长处理时间。合理调整这些参数可以在不影响系统整体响应性的前提下，提高网络处理能力。

```C
net.core.netdev_budget = 300
net.core.netdev_budget_usecs = 8000
```

我通过调大上述参数，解决了一个生产环境丢包问题。当然完整的丢包分析，需要覆盖驱动层到协议层的整个链路，找到丢包点再进行优化。

**注：如下是Linux内核网络学习记录，后续根据脉络再进行整理。**

## 5. Mellanox驱动

Mellanox网卡驱动作为PCIe设备驱动，通过`pci_driver`结构体定义了与内核PCI子系统的接口。当系统检测到Mellanox网卡时，PCI子系统会调用驱动的`probe`函数（`init_one`）进行设备初始化。这个初始化过程包括硬件资源分配、DMA映射建立、中断注册、队列创建等关键步骤。其中，ring buffer的分配是保证高性能网络传输的基础，它为网卡和内核之间的数据交换提供了高效的缓冲机制。

```C
static struct pci_driver mlx5_core_driver = {
	.name           = DRIVER_NAME,
	.id_table       = mlx5_core_pci_table,
	.probe          = init_one,
	.remove         = remove_one,
	.shutdown	= shutdown,
	.err_handler	= &mlx5_err_handler,
	.sriov_configure   = mlx5_core_sriov_configure,
};
```

### 5.1 mlx5分配ring buffer

Ring buffer的分配过程展示了Mellanox驱动如何为高性能网络传输准备内存资源。整个过程从设备初始化开始，通过创建Event Queue（EQ）来处理各种硬件事件。EQ的大小经过精心计算，考虑了基本大小和额外缓冲，并通过`roundup_pow_of_two`确保大小是2的幂次，这有利于硬件优化。最终通过DMA一致性内存分配函数获取物理连续的内存区域，这种内存可以被网卡硬件直接访问，避免了数据复制的开销。

```plantuml
@startuml
:init_one();
:mlx5_load_one();
:alloc_comp_eqs();
:mlx5_create_map_eq();
:mlx5_buf_alloc();
:mlx5_buf_alloc_node();
split
 :kzalloc();
 note left
  `mlx5_buf_list`指针
 end note
split again
 :mlx5_dma_zalloc_coherent_node();
 note right
  分配DMA内存，即ring buffer
 end note
 :dma_zalloc_coherent();
 :dma_alloc_coherent();
end split
@enduml
```

**注意**：经过实际测试，发现调整ring buffer会增加内存开销，具体取决于每个驱动的实现、PAGESIZE、MTU。

**场景**：128核CPU，512G内存，mlx5_core 5.8-3.0.7驱动，MTU 9000，PAGESIZE 4096，queue数量63。

**结论**：将一个网卡的ring buffer从1024调整到8192大约增加6G内存，2个网卡增加12G内存。

**评估公式**：`PAGESIZE*page_count*buffer_count*queue_count`。PAGESIZE影响最小分配内存，MTU影响page个数。

如下以4k的PAGESIZE和9000MTU为例，ring buffer从1024调整到8192进行计算。
- `4*3*(8192-1024)*63/1024.0=5292M`
- `4*4*(8192-1024)*63/1024.0=7056M`


分配**ring buffer**的核心逻辑主要在`alloc_comp_eqs()`中。

1. 为每个完成队列分配一个CPU中断映射表
2. 将中断向量添加到CPU中断映射中
3. 使用snprintf生成完成队列的名称，并调用mlx5_create_map_eq函数创建完成队列

```C
enum {
	MLX5_COMP_EQ_SIZE = 1024,
};
```

```C

static int alloc_comp_eqs(struct mlx5_core_dev *dev)
{
	struct mlx5_eq_table *table = &dev->priv.eq_table;
	char name[MLX5_MAX_IRQ_NAME];
	struct mlx5_eq *eq;
	int ncomp_vec;
	int nent;
	int err;
	int i;

	INIT_LIST_HEAD(&table->comp_eqs_list);
	ncomp_vec = table->num_comp_vectors;
	nent = MLX5_COMP_EQ_SIZE;
#ifdef CONFIG_RFS_ACCEL
	dev->rmap = alloc_irq_cpu_rmap(ncomp_vec);
	if (!dev->rmap)
		return -ENOMEM;
#endif
	for (i = 0; i < ncomp_vec; i++) {
		eq = kzalloc(sizeof(*eq), GFP_KERNEL);
		if (!eq) {
			err = -ENOMEM;
			goto clean;
		}

#ifdef CONFIG_RFS_ACCEL
		irq_cpu_rmap_add(dev->rmap, pci_irq_vector(dev->pdev,
				 MLX5_EQ_VEC_COMP_BASE + i));
#endif
		snprintf(name, MLX5_MAX_IRQ_NAME, "mlx5_comp%d", i);
		err = mlx5_create_map_eq(dev, eq,
					 i + MLX5_EQ_VEC_COMP_BASE, nent, 0,
					 name, MLX5_EQ_TYPE_COMP);
		if (err) {
			kfree(eq);
			goto clean;
		}
		mlx5_core_dbg(dev, "allocated completion EQN %d\n", eq->eqn);
		eq->index = i;
		spin_lock(&table->lock);
		list_add_tail(&eq->list, &table->comp_eqs_list);
		spin_unlock(&table->lock);
	}

	return 0;

clean:
	free_comp_eqs(dev);
	return err;
}
```
{collapsible="true" collapsed-title="alloc_comp_eqs" default-state="collapsed"}

在`alloc_comp_eqs()`中执行了如下代码分配内存，决定了后续函数分配内存的大小。
```C
	nent = MLX5_COMP_EQ_SIZE;
	// 省略
		err = mlx5_create_map_eq(dev, eq,
					 i + MLX5_EQ_VEC_COMP_BASE, nent, 0,
					 name, MLX5_EQ_TYPE_COMP);
```

```C
enum {
	MLX5_NUM_SPARE_EQE	= 0x80, // 十进制为128
	MLX5_NUM_ASYNC_EQE	= 0x1000,
	MLX5_NUM_CMD_EQE	= 32,
	MLX5_NUM_PF_DRAIN	= 64,
};
```

```C
enum {
	MLX5_EQE_SIZE		= sizeof(struct mlx5_eqe),
	MLX5_EQE_OWNER_INIT_VAL	= 0x1,
};
```

```C
union ev_data {
	__be32				raw[7];
	struct mlx5_eqe_cmd		cmd;
	struct mlx5_eqe_comp		comp;
	struct mlx5_eqe_qp_srq		qp_srq;
	struct mlx5_eqe_cq_err		cq_err;
	struct mlx5_eqe_port_state	port;
	struct mlx5_eqe_gpio		gpio;
	struct mlx5_eqe_congestion	cong;
	struct mlx5_eqe_stall_vl	stall_vl;
	struct mlx5_eqe_page_req	req_pages;
	struct mlx5_eqe_page_fault	page_fault;
	struct mlx5_eqe_vport_change	vport_change;
	struct mlx5_eqe_port_module	port_module;
	struct mlx5_eqe_pps		pps;
	struct mlx5_eqe_dct             dct;
	struct mlx5_eqe_temp_warning	temp_warning;
} __packed;
```
{collapsible="true" collapsed-title="ev_data" default-state="collapsed"}

```C
struct mlx5_eqe {
	u8		rsvd0;
	u8		type;
	u8		rsvd1;
	u8		sub_type;
	__be32		rsvd2[7];
	union ev_data	data;
	__be16		rsvd3;
	u8		signature;
	u8		owner;
} __packed;
```
{collapsible="true" collapsed-title="mlx5_eqe" default-state="collapsed"}

`mlx5_create_map_eq`函数是创建和映射Event Queue的核心函数。它不仅负责分配内存，还要完成硬件配置、中断注册和软件初始化等多项工作。函数首先计算EQ的实际大小（基础大小加上备用空间），然后分配DMA内存。接着通过固件命令创建硬件EQ，并注册中断处理函数。最后，根据EQ类型初始化相应的处理机制，如tasklet用于普通完成事件处理。

```C
int mlx5_create_map_eq(struct mlx5_core_dev *dev, struct mlx5_eq *eq, u8 vecidx,
		       int nent, u64 mask, const char *name,
		       enum mlx5_eq_type type)
{
	struct mlx5_cq_table *cq_table = &eq->cq_table;
	u32 out[MLX5_ST_SZ_DW(create_eq_out)] = {0};
	struct mlx5_priv *priv = &dev->priv;
	irq_handler_t handler;
	__be64 *pas;
	void *eqc;
	int inlen;
	u32 *in;
	int err;

	/* Init CQ table */
	memset(cq_table, 0, sizeof(*cq_table));
	spin_lock_init(&cq_table->lock);
	INIT_RADIX_TREE(&cq_table->tree, GFP_ATOMIC);

	eq->type = type;
	// 1024+128再roundup_pow_of_two后得到2048
	eq->nent = roundup_pow_of_two(nent + MLX5_NUM_SPARE_EQE);
	eq->cons_index = 0;
	// 2048*64=131072字节
	err = mlx5_buf_alloc(dev, eq->nent * MLX5_EQE_SIZE, &eq->buf);
	if (err)
		return err;

#ifdef CONFIG_INFINIBAND_ON_DEMAND_PAGING
	if (type == MLX5_EQ_TYPE_PF)
		handler = mlx5_eq_pf_int;
	else
#endif
		handler = mlx5_eq_int;

	init_eq_buf(eq);

	inlen = MLX5_ST_SZ_BYTES(create_eq_in) +
		MLX5_FLD_SZ_BYTES(create_eq_in, pas[0]) * eq->buf.npages;

	in = kvzalloc(inlen, GFP_KERNEL);
	if (!in) {
		err = -ENOMEM;
		goto err_buf;
	}

	pas = (__be64 *)MLX5_ADDR_OF(create_eq_in, in, pas);
	mlx5_fill_page_array(&eq->buf, pas);

	MLX5_SET(create_eq_in, in, opcode, MLX5_CMD_OP_CREATE_EQ);
	MLX5_SET64(create_eq_in, in, event_bitmask, mask);

	eqc = MLX5_ADDR_OF(create_eq_in, in, eq_context_entry);
	MLX5_SET(eqc, eqc, log_eq_size, ilog2(eq->nent));
	MLX5_SET(eqc, eqc, uar_page, priv->uar->index);
	MLX5_SET(eqc, eqc, intr, vecidx);
	MLX5_SET(eqc, eqc, log_page_size,
		 eq->buf.page_shift - MLX5_ADAPTER_PAGE_SHIFT);

	err = mlx5_cmd_exec(dev, in, inlen, out, sizeof(out));
	if (err)
		goto err_in;

	snprintf(priv->irq_info[vecidx].name, MLX5_MAX_IRQ_NAME, "%s@pci:%s",
		 name, pci_name(dev->pdev));

	eq->eqn = MLX5_GET(create_eq_out, out, eq_number);
	eq->irqn = pci_irq_vector(dev->pdev, vecidx);
	eq->dev = dev;
	eq->doorbell = priv->uar->map + MLX5_EQ_DOORBEL_OFFSET;
	err = request_irq(eq->irqn, handler, 0,
			  priv->irq_info[vecidx].name, eq);
	if (err)
		goto err_eq;

	err = mlx5_debug_eq_add(dev, eq);
	if (err)
		goto err_irq;

#ifdef CONFIG_INFINIBAND_ON_DEMAND_PAGING
	if (type == MLX5_EQ_TYPE_PF) {
		err = init_pf_ctx(&eq->pf_ctx, name);
		if (err)
			goto err_irq;
	} else
#endif
	{
		INIT_LIST_HEAD(&eq->tasklet_ctx.list);
		INIT_LIST_HEAD(&eq->tasklet_ctx.process_list);
		spin_lock_init(&eq->tasklet_ctx.lock);
		tasklet_init(&eq->tasklet_ctx.task, mlx5_cq_tasklet_cb,
			     (unsigned long)&eq->tasklet_ctx);
	}

	/* EQs are created in ARMED state
	 */
	eq_update_ci(eq, 1);

	kvfree(in);
	return 0;

err_irq:
	free_irq(eq->irqn, eq);

err_eq:
	mlx5_cmd_destroy_eq(dev, eq->eqn);

err_in:
	kvfree(in);

err_buf:
	mlx5_buf_free(dev, &eq->buf);
	return err;
}
```
{collapsible="true" collapsed-title="mlx5_create_map_eq" default-state="collapsed"}

内存分配的精确计算体现在以下代码片段中。通过将基础EQ大小（1024）加上备用空间（128），再进行2的幂次对齐，最终得到2048个条目。每个条目64字节，总共需要131072字节（128KB）的DMA内存。这种对齐策略不仅有利于硬件访问优化，还简化了索引计算。

```C
	eq->type = type;
	// 1024+128再roundup_pow_of_two后得到2048
	eq->nent = roundup_pow_of_two(nent + MLX5_NUM_SPARE_EQE);
	eq->cons_index = 0;
	// 2048*64=131072字节
	err = mlx5_buf_alloc(dev, eq->nent * MLX5_EQE_SIZE, &eq->buf);
	if (err)
		return err;
```

```C
// /home/kangxiaoning/workspace/kernel-4.19.90-2404.2.0/drivers/net/ethernet/mellanox/mlx5/core/alloc.c

int mlx5_buf_alloc_node(struct mlx5_core_dev *dev, int size,
			struct mlx5_frag_buf *buf, int node)
{
	dma_addr_t t;

	buf->size = size;
	buf->npages       = 1;
	buf->page_shift   = (u8)get_order(size) + PAGE_SHIFT;

	buf->frags = kzalloc(sizeof(*buf->frags), GFP_KERNEL);
	if (!buf->frags)
		return -ENOMEM;

	buf->frags->buf   = mlx5_dma_zalloc_coherent_node(dev, size,
							  &t, node);
	if (!buf->frags->buf)
		goto err_out;

	buf->frags->map = t;

	while (t & ((1 << buf->page_shift) - 1)) {
		--buf->page_shift;
		buf->npages *= 2;
	}

	return 0;
err_out:
	kfree(buf->frags);
	return -ENOMEM;
}
```
{collapsible="true" collapsed-title="mlx5_buf_alloc_node" default-state="collapsed"}

从下面的收包逻辑可以判断，分配ring buffer过程中把RX的skb空间也分配了，收到包以后就从队列中取一个skb进行处理。

```C
void mlx5e_handle_rx_cqe(struct mlx5e_rq *rq, struct mlx5_cqe64 *cqe)
{
	struct mlx5_wq_cyc *wq = &rq->wqe.wq;
	struct mlx5e_wqe_frag_info *wi;
	struct sk_buff *skb;
	u32 cqe_bcnt;
	u16 ci;

	ci       = mlx5_wq_cyc_ctr2ix(wq, be16_to_cpu(cqe->wqe_counter));
	wi       = get_frag(rq, ci);
	cqe_bcnt = be32_to_cpu(cqe->byte_cnt);

	skb = rq->wqe.skb_from_cqe(rq, cqe, wi, cqe_bcnt);
	if (!skb) {
		/* probably for XDP */
		if (__test_and_clear_bit(MLX5E_RQ_FLAG_XDP_XMIT, rq->flags)) {
			/* do not return page to cache,
			 * it will be returned on XDP_TX completion.
			 */
			goto wq_cyc_pop;
		}
		goto free_wqe;
	}

	mlx5e_complete_rx_cqe(rq, cqe, cqe_bcnt, skb);
	napi_gro_receive(rq->cq.napi, skb);

free_wqe:
	mlx5e_free_rx_wqe(rq, wi, true);
wq_cyc_pop:
	mlx5_wq_cyc_pop(wq);
}
```
{collapsible="true" collapsed-title="mlx5e_handle_rx_cqe" default-state="collapsed"}

### 5.2 mlx5初始化tasklet

```C
int mlx5_create_map_eq(struct mlx5_core_dev *dev, struct mlx5_eq *eq, u8 vecidx,
		       int nent, u64 mask, const char *name,
		       enum mlx5_eq_type type)
{
	struct mlx5_cq_table *cq_table = &eq->cq_table;
	u32 out[MLX5_ST_SZ_DW(create_eq_out)] = {0};
	struct mlx5_priv *priv = &dev->priv;
	irq_handler_t handler;
	__be64 *pas;
	void *eqc;
	int inlen;
	u32 *in;
	int err;

	/* Init CQ table */
	memset(cq_table, 0, sizeof(*cq_table));
	spin_lock_init(&cq_table->lock);
	INIT_RADIX_TREE(&cq_table->tree, GFP_ATOMIC);

	eq->type = type;
	eq->nent = roundup_pow_of_two(nent + MLX5_NUM_SPARE_EQE);
	eq->cons_index = 0;
	err = mlx5_buf_alloc(dev, eq->nent * MLX5_EQE_SIZE, &eq->buf);
	if (err)
		return err;

#ifdef CONFIG_INFINIBAND_ON_DEMAND_PAGING
	if (type == MLX5_EQ_TYPE_PF)
		handler = mlx5_eq_pf_int;
	else
#endif
		handler = mlx5_eq_int;

	init_eq_buf(eq);

	inlen = MLX5_ST_SZ_BYTES(create_eq_in) +
		MLX5_FLD_SZ_BYTES(create_eq_in, pas[0]) * eq->buf.npages;

	in = kvzalloc(inlen, GFP_KERNEL);
	if (!in) {
		err = -ENOMEM;
		goto err_buf;
	}

	pas = (__be64 *)MLX5_ADDR_OF(create_eq_in, in, pas);
	mlx5_fill_page_array(&eq->buf, pas);

	MLX5_SET(create_eq_in, in, opcode, MLX5_CMD_OP_CREATE_EQ);
	MLX5_SET64(create_eq_in, in, event_bitmask, mask);

	eqc = MLX5_ADDR_OF(create_eq_in, in, eq_context_entry);
	MLX5_SET(eqc, eqc, log_eq_size, ilog2(eq->nent));
	MLX5_SET(eqc, eqc, uar_page, priv->uar->index);
	MLX5_SET(eqc, eqc, intr, vecidx);
	MLX5_SET(eqc, eqc, log_page_size,
		 eq->buf.page_shift - MLX5_ADAPTER_PAGE_SHIFT);

	err = mlx5_cmd_exec(dev, in, inlen, out, sizeof(out));
	if (err)
		goto err_in;

	snprintf(priv->irq_info[vecidx].name, MLX5_MAX_IRQ_NAME, "%s@pci:%s",
		 name, pci_name(dev->pdev));

	eq->eqn = MLX5_GET(create_eq_out, out, eq_number);
	eq->irqn = pci_irq_vector(dev->pdev, vecidx);
	eq->dev = dev;
	eq->doorbell = priv->uar->map + MLX5_EQ_DOORBEL_OFFSET;
	err = request_irq(eq->irqn, handler, 0,
			  priv->irq_info[vecidx].name, eq);
	if (err)
		goto err_eq;

	err = mlx5_debug_eq_add(dev, eq);
	if (err)
		goto err_irq;

#ifdef CONFIG_INFINIBAND_ON_DEMAND_PAGING
	if (type == MLX5_EQ_TYPE_PF) {
		err = init_pf_ctx(&eq->pf_ctx, name);
		if (err)
			goto err_irq;
	} else
#endif
	{
		INIT_LIST_HEAD(&eq->tasklet_ctx.list);
		INIT_LIST_HEAD(&eq->tasklet_ctx.process_list);
		spin_lock_init(&eq->tasklet_ctx.lock);
		tasklet_init(&eq->tasklet_ctx.task, mlx5_cq_tasklet_cb,
			     (unsigned long)&eq->tasklet_ctx);
	}

	/* EQs are created in ARMED state
	 */
	eq_update_ci(eq, 1);

	kvfree(in);
	return 0;

err_irq:
	free_irq(eq->irqn, eq);

err_eq:
	mlx5_cmd_destroy_eq(dev, eq->eqn);

err_in:
	kvfree(in);

err_buf:
	mlx5_buf_free(dev, &eq->buf);
	return err;
}
```
{collapsible="true" collapsed-title="mlx5_create_map_eq" default-state="collapsed"}

## 6. Linux kernel

Linux内核网络栈通过回调函数体系实现了协议的模块化和可扩展性。这些回调函数覆盖了从套接字操作到协议处理的各个层面，使得不同的网络协议可以共享通用的框架代码，同时保持各自的特定实现。

### 6.1 Socket相关回调

套接字层回调函数定义了用户空间程序与内核网络栈的接口。`inetsw_array`数组注册了不同类型套接字（TCP、UDP、RAW等）的处理函数。每种套接字类型都有对应的`proto`结构（协议处理）和`proto_ops`结构（套接字操作），分别处理协议相关逻辑和用户接口操作。`socket_file_ops`则将套接字抽象为文件，使得网络编程可以使用统一的文件操作接口。

```C
static struct inet_protosw inetsw_array[] =
{
	{
		.type =       SOCK_STREAM,
		.protocol =   IPPROTO_TCP,
		.prot =       &tcp_prot,
		.ops =        &inet_stream_ops,
		.flags =      INET_PROTOSW_PERMANENT |
			      INET_PROTOSW_ICSK,
	},

	{
		.type =       SOCK_DGRAM,
		.protocol =   IPPROTO_UDP,
		.prot =       &udp_prot,
		.ops =        &inet_dgram_ops,
		.flags =      INET_PROTOSW_PERMANENT,
       },

       {
		.type =       SOCK_DGRAM,
		.protocol =   IPPROTO_ICMP,
		.prot =       &ping_prot,
		.ops =        &inet_sockraw_ops,
		.flags =      INET_PROTOSW_REUSE,
       },

       {
	       .type =       SOCK_RAW,
	       .protocol =   IPPROTO_IP,	/* wild card */
	       .prot =       &raw_prot,
	       .ops =        &inet_sockraw_ops,
	       .flags =      INET_PROTOSW_REUSE,
       }
};
```
{collapsible="true" collapsed-title="inetsw_array[]" default-state="collapsed"}

`tcp_prot`结构体定义了TCP协议的核心操作函数，包括连接建立、数据收发、连接关闭等。这些函数实现了TCP的可靠传输机制，包括序列号管理、确认机制、流量控制和拥塞控制。通过内存压力回调和孤儿套接字管理，TCP协议栈可以在系统资源紧张时采取适当的措施，保证系统稳定性。

```C
struct proto tcp_prot = {
	.name			= "TCP",
	.owner			= THIS_MODULE,
	.close			= tcp_close,
	.pre_connect		= tcp_v4_pre_connect,
	.connect		= tcp_v4_connect,
	.disconnect		= tcp_disconnect,
	.accept			= inet_csk_accept,
	.ioctl			= tcp_ioctl,
	.init			= tcp_v4_init_sock,
	.destroy		= tcp_v4_destroy_sock,
	.shutdown		= tcp_shutdown,
	.setsockopt		= tcp_setsockopt,
	.getsockopt		= tcp_getsockopt,
	.keepalive		= tcp_set_keepalive,
	.recvmsg		= tcp_recvmsg,
	.sendmsg		= tcp_sendmsg,
	.sendpage		= tcp_sendpage,
	.backlog_rcv		= tcp_v4_do_rcv,
	.release_cb		= tcp_release_cb,
	.hash			= inet_hash,
	.unhash			= inet_unhash,
	.get_port		= inet_csk_get_port,
	.enter_memory_pressure	= tcp_enter_memory_pressure,
	.leave_memory_pressure	= tcp_leave_memory_pressure,
	.stream_memory_free	= tcp_stream_memory_free,
	.sockets_allocated	= &tcp_sockets_allocated,
	.orphan_count		= &tcp_orphan_count,
	.memory_allocated	= &tcp_memory_allocated,
	.memory_pressure	= &tcp_memory_pressure,
	.sysctl_mem		= sysctl_tcp_mem,
	.sysctl_wmem_offset	= offsetof(struct net, ipv4.sysctl_tcp_wmem),
	.sysctl_rmem_offset	= offsetof(struct net, ipv4.sysctl_tcp_rmem),
	.max_header		= MAX_TCP_HEADER,
	.obj_size		= sizeof(struct tcp_sock),
	.slab_flags		= SLAB_TYPESAFE_BY_RCU,
	.twsk_prot		= &tcp_timewait_sock_ops,
	.rsk_prot		= &tcp_request_sock_ops,
	.h.hashinfo		= &tcp_hashinfo,
	.no_autobind		= true,
#ifdef CONFIG_COMPAT
	.compat_setsockopt	= compat_tcp_setsockopt,
	.compat_getsockopt	= compat_tcp_getsockopt,
#endif
	.diag_destroy		= tcp_abort,
};
```
{collapsible="true" collapsed-title="tcp_prot" default-state="collapsed"}

`inet_stream_ops`提供了面向流套接字的操作接口，这些函数直接对应到用户空间的系统调用。通过`tcp_poll`实现了高效的I/O多路复用，`tcp_mmap`支持零拷贝优化，`tcp_splice_read`实现了高效的数据转发。

```C
const struct proto_ops inet_stream_ops = {
	.family		   = PF_INET,
	.owner		   = THIS_MODULE,
	.release	   = inet_release,
	.bind		   = inet_bind,
	.connect	   = inet_stream_connect,
	.socketpair	   = sock_no_socketpair,
	.accept		   = inet_accept,
	.getname	   = inet_getname,
	.poll		   = tcp_poll,
	.ioctl		   = inet_ioctl,
	.listen		   = inet_listen,
	.shutdown	   = inet_shutdown,
	.setsockopt	   = sock_common_setsockopt,
	.getsockopt	   = sock_common_getsockopt,
	.sendmsg	   = inet_sendmsg,
	.recvmsg	   = inet_recvmsg,
#ifdef CONFIG_MMU
	.mmap		   = tcp_mmap,
#endif
	.sendpage	   = inet_sendpage,
	.splice_read	   = tcp_splice_read,
	.read_sock	   = tcp_read_sock,
	.sendmsg_locked    = tcp_sendmsg_locked,
	.sendpage_locked   = tcp_sendpage_locked,
	.peek_len	   = tcp_peek_len,
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_sock_common_setsockopt,
	.compat_getsockopt = compat_sock_common_getsockopt,
	.compat_ioctl	   = inet_compat_ioctl,
#endif
	.set_rcvlowat	   = tcp_set_rcvlowat,
};
```
{collapsible="true" collapsed-title="inet_stream_ops" default-state="collapsed"}

套接字文件操作接口使得网络编程可以像操作普通文件一样操作网络连接。通过VFS层的抽象，应用程序可以使用read/write、poll/select等标准接口进行网络通信，大大简化了网络编程的复杂性。

```C
static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
};
```
{collapsible="true" collapsed-title="socket_file_ops" default-state="collapsed"}

### 6.2 TCP相关回调

TCP请求处理回调函数管理TCP连接建立过程中的各种状态。`tcp_request_sock_ops`处理SYN包的接收、SYN+ACK的发送和重传逻辑。这些函数实现了TCP的三次握手机制，包括SYN cookie等防御机制，保护服务器免受SYN flood攻击。

```C
struct request_sock_ops tcp_request_sock_ops __read_mostly = {
	.family		=	PF_INET,
	.obj_size	=	sizeof(struct tcp_request_sock),
	.rtx_syn_ack	=	tcp_rtx_synack,
	.send_ack	=	tcp_v4_reqsk_send_ack,
	.destructor	=	tcp_v4_reqsk_destructor,
	.send_reset	=	tcp_v4_send_reset,
	.syn_ack_timeout =	tcp_syn_ack_timeout,
};
```
{collapsible="true" collapsed-title="tcp_request_sock_ops" default-state="collapsed"}

IPv4特定的TCP请求处理函数提供了更细粒度的控制。包括MSS（Maximum Segment Size）计算、初始序列号生成、时间戳偏移初始化等。MD5签名支持增强了TCP连接的安全性，常用于BGP等关键协议。

```C
const struct tcp_request_sock_ops tcp_request_sock_ipv4_ops = {
	.mss_clamp	=	TCP_MSS_DEFAULT,
#ifdef CONFIG_TCP_MD5SIG
	.req_md5_lookup	=	tcp_v4_md5_lookup,
	.calc_md5_hash	=	tcp_v4_md5_hash_skb,
#endif
	.init_req	=	tcp_v4_init_req,
#ifdef CONFIG_SYN_COOKIES
	.cookie_init_seq =	cookie_v4_init_sequence,
#endif
	.route_req	=	tcp_v4_route_req,
	.init_seq	=	tcp_v4_init_seq,
	.init_ts_off	=	tcp_v4_init_ts_off,
	.send_synack	=	tcp_v4_send_synack,
};
```
{collapsible="true" collapsed-title="tcp_request_sock_ipv4_ops" default-state="collapsed"}

### 6.3 IP相关回调

TCP套接字初始化函数设置了IPv4特定的操作函数。这种分层设计使得TCP协议栈可以同时支持IPv4和IPv6，通过不同的`af_ops`实现协议特定的功能，如路由查找、校验和计算等。

```C
static int tcp_v4_init_sock(struct sock *sk)
{
	struct inet_connection_sock *icsk = inet_csk(sk);

	tcp_init_sock(sk);

	icsk->icsk_af_ops = &ipv4_specific;

#ifdef CONFIG_TCP_MD5SIG
	tcp_sk(sk)->af_specific = &tcp_sock_ipv4_specific;
#endif

	return 0;
}
```
{collapsible="true" collapsed-title="tcp_v4_init_sock" default-state="collapsed"}

`ipv4_specific`结构体包含了IPv4协议族特定的连接操作。这些函数处理IP层的各种功能，包括IP包的排队发送、TCP校验和计算、路由缓存管理等。通过这种抽象，TCP协议栈可以透明地支持不同的网络层协议。

```C
const struct inet_connection_sock_af_ops ipv4_specific = {
	.queue_xmit	   = ip_queue_xmit,
	.send_check	   = tcp_v4_send_check,
	.rebuild_header	   = inet_sk_rebuild_header,
	.sk_rx_dst_set	   = inet_sk_rx_dst_set,
	.conn_request	   = tcp_v4_conn_request,
	.syn_recv_sock	   = tcp_v4_syn_recv_sock,
	.net_header_len	   = sizeof(struct iphdr),
	.setsockopt	   = ip_setsockopt,
	.getsockopt	   = ip_getsockopt,
	.addr2sockaddr	   = inet_csk_addr2sockaddr,
	.sockaddr_len	   = sizeof(struct sockaddr_in),
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_ip_setsockopt,
	.compat_getsockopt = compat_ip_getsockopt,
#endif
	.mtu_reduced	   = tcp_v4_mtu_reduced,
};
```
{collapsible="true" collapsed-title="ipv4_specific" default-state="collapsed"}

### 6.4 Neighbour相关回调

邻居子系统（Neighbour Subsystem）负责管理链路层地址解析，在IPv4中主要是ARP协议。不同的邻居操作结构对应不同的使用场景：`arp_generic_ops`用于需要地址解析的普通情况，`arp_hh_ops`优化了硬件头部缓存的情况，`arp_direct_ops`用于点对点链路等不需要地址解析的场景。

```C
static const struct neigh_ops arp_generic_ops = {
	.family =		AF_INET,
	.solicit =		arp_solicit,
	.error_report =		arp_error_report,
	.output =		neigh_resolve_output,
	.connected_output =	neigh_connected_output,
};

static const struct neigh_ops arp_hh_ops = {
	.family =		AF_INET,
	.solicit =		arp_solicit,
	.error_report =		arp_error_report,
	.output =		neigh_resolve_output,
	.connected_output =	neigh_resolve_output,
};

static const struct neigh_ops arp_direct_ops = {
	.family =		AF_INET,
	.output =		neigh_direct_output,
	.connected_output =	neigh_direct_output,
};
```

ARP表结构定义了ARP协议的各种参数和行为。包括ARP表项的老化时间、重传间隔、队列长度等。这些参数经过精心调优，在地址解析的及时性和网络资源消耗之间取得平衡。通过垃圾回收机制，系统可以自动清理过期的ARP表项，防止内存泄漏。
```C
struct neigh_table arp_tbl = {
	.family		= AF_INET,
	.key_len	= 4,
	.protocol	= cpu_to_be16(ETH_P_IP),
	.hash		= arp_hash,
	.key_eq		= arp_key_eq,
	.constructor	= arp_constructor,
	.proxy_redo	= parp_redo,
	.id		= "arp_cache",
	.parms		= {
		.tbl			= &arp_tbl,
		.reachable_time		= 30 * HZ,
		.data	= {
			[NEIGH_VAR_MCAST_PROBES] = 3,
			[NEIGH_VAR_UCAST_PROBES] = 3,
			[NEIGH_VAR_RETRANS_TIME] = 1 * HZ,
			[NEIGH_VAR_BASE_REACHABLE_TIME] = 30 * HZ,
			[NEIGH_VAR_DELAY_PROBE_TIME] = 5 * HZ,
			[NEIGH_VAR_GC_STALETIME] = 60 * HZ,
			[NEIGH_VAR_QUEUE_LEN_BYTES] = SK_WMEM_MAX,
			[NEIGH_VAR_PROXY_QLEN] = 64,
			[NEIGH_VAR_ANYCAST_DELAY] = 1 * HZ,
			[NEIGH_VAR_PROXY_DELAY]	= (8 * HZ) / 10,
			[NEIGH_VAR_LOCKTIME] = 1 * HZ,
		},
	},
	.gc_interval	= 30 * HZ,
	.gc_thresh1	= 128,
	.gc_thresh2	= 512,
	.gc_thresh3	= 1024,
};
EXPORT_SYMBOL(arp_tbl);
```
{collapsible="true" collapsed-title="arp_tbl" default-state="collapsed"}

## 总结

`time_squeeze`作为一个重要的性能指标，直接反映了软中断处理网络数据包的效率。当这个指标持续增长时，表明系统在分配的预算内无法及时处理所有网络数据包，存在潜在的性能瓶颈。通过调整`netdev_budget`和`netdev_budget_usecs`参数，可以有效缓解这一问题，提高网络处理能力。

内容写的比较杂，主要发现包括：

1. **软中断处理机制**：Linux内核通过NAPI机制实现了高效的批量数据包处理，通过预算控制避免单个网卡垄断CPU资源，保证了系统的公平性和响应性。

2. **驱动实现细节**：Mellanox驱动展示了现代高性能网卡驱动的设计模式，包括多队列支持、动态中断调节、预分配缓冲区等优化技术。

3. **内存管理策略**：Ring buffer的分配需要考虑多个因素，包括队列数量、MTU大小、页面大小等。合理的内存规划对系统性能至关重要。

4. **性能优化方向**：除了调整内核参数外，还可以通过CPU亲和性设置、中断分配优化、队列数量调整等手段进一步提升网络性能。

5. **监控和诊断**：`/proc/net/softnet_stat`提供了丰富的性能监控数据，结合其他工具可以全面了解系统的网络处理状态。

在实际生产环境中，网络性能优化需要综合考虑硬件能力、软件配置和业务特征。通过持续监控关键指标，及时发现并解决性能瓶颈，才能确保系统在高负载下稳定运行。本文提供的分析方法和优化建议，可以作为网络性能调优的参考，更好地理解和优化Linux网络栈的性能表现。
