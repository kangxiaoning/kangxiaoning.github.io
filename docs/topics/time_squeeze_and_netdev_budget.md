# time_squeeze和netdev_budget

<show-structure depth="3"/>

time_squeeze持续增长，说明`softirq`的收包预算用完时`TX/RX queue`中仍然有包等待处理。

只要**TX/RX queue**在下次`softirq`处理之前没有**overflow**，那就不会因为`time_squeeze`增加而导致丢包，但如果`time_squeeze`持续增加，就有`TX/RX queue`溢出导致丢包的可能，需要结合**packet_drop**指标来看。

## 1. /proc/net/softnet_stat

通过如下代码可知对应列的含义。

- Column-01: **packet_process**: the number of frames received by the interrupt handler.
- Column-02: **packet_drop**: the number of frames dropped due to netdev_max_backlog being exceeded.
- Column-03: **time_squeeze**: the number of times ksoftirqd ran out of netdev_budget or CPU time when there was still work to be done.
- Column-09: **cpu_collision**: collision occur while obtaining device lock while transmitting.
- Column-10: **received_rps**: number of times cpu woken up received_rps.
- Column-11: **flow_limit_count**: number of times reached flow limit count.

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

## 3. Mellanox驱动poll方法

在Mellanox驱动代码中搜索`netif_napi_add`关键字即可找到，如下是搜索结果。

```C
	netif_napi_add(netdev, &c->napi, mlx5e_napi_poll, 64);
```

`netif_napi_add()`函数定义如下。
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

在`net_rx_action()`函数中，`budget -= napi_poll(n, &repoll);`最终会执行到网卡驱动的`poll()`方法。

如下是Mellanox驱动对应的`poll()`方法，可以看到不光执行了**rx**方向的收包操作，还执行了**tx**方向的发包操作，因此如果time_squeeze持续增长，说明收包和发包都可能出现堆积或者丢包。

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
		busy |= mlx5e_poll_tx_cq(&c->sq[i].cq, budget);

	busy |= mlx5e_poll_xdpsq_cq(&c->xdpsq.cq);

	if (c->xdp)
		busy |= mlx5e_poll_xdpsq_cq(&c->rq.xdpsq.cq);

	if (likely(budget)) { /* budget=0 means: don't poll rx rings */
		work_done = mlx5e_poll_rx_cq(&c->rq.cq, budget);
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

参考：[](https://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)

## 4. 参数优化

可以调整如下参数，然后继续观察time_squeeze变化情况。

```C
net.core.netdev_budget = 300
net.core.netdev_budget_usecs = 8000
```