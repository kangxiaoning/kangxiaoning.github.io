# Linux Neighbor源码分析

<show-structure depth="2"/>

本文基于Linux内核3.10源码分析IP层数据包处理、邻居表管理。


## 1. 什么时候需要创建Neighbor条目

在Linux内核中，邻居表（Neighbor Table）是网络子系统中的一个核心组件，它维护着IP地址到MAC地址的映射关系，类似于ARP缓存的功能。当网络数据包需要通过ARP（地址解析协议）或其他对应的协议（如IPv6下的NDP，Neighbor Discovery Protocol）来解析目标主机的硬件地址时，会创建neighbour条目(也就是ARP条目)，比如下列情况。

1. 发送数据包至本地网络上的未知MAC地址时，当IP层需要将一个IP数据包发送到同一链路上的一个目的IP地址，但不知道该目的IP对应的MAC地址时，内核会触发ARP请求以获取目标MAC地址，并且在这个过程中创建或更新neighbour条目。
2. 接收ARP/NDP响应时，当收到ARP应答或者NDP中的Neighbor Advertisement消息，其中包含了其他主机的IP与MAC地址对应关系时，内核会根据这些信息创建或更新neighbour条目。
3. 静态配置，可以通过命令行工具、配置文件等方式手动为特定的IP地址静态配置其对应的MAC地址，此时也会在neighbour表中创建相应的条目。
4. 路由更新时，当路由表发生变更，导致新的下一跳成为本地链路时，对于新的下一跳地址，如果之前没有对应的neighbour条目，则会触发ARP查询并创建新条目。
5. GARP(Gratuitous ARP)或者NS(Neighbor Solicitation)，接收到Gratuitous ARP或NDP的NS消息时，即使不是直接发给本机的，也可能用于更新或创建neighbour条目。

一句话总结，每当Linux需要或者从网络上获得到本地链路上的IP与MAC地址映射关系时，都会在neighbour cache（即ARP缓存或NDP缓存）中创建或更新相应条目。

## 2. IP层查找路由时更新neighbor条目

在Linux网络协议栈中，当一个IP数据包准备从网络层传递到数据链路层时，需要完成两个关键任务：
1. **确定下一跳地址**：根据路由表查找数据包应该发往哪个下一跳。
2. **解析物理地址**：将下一跳的IP地址转换为对应的MAC地址。

`ip_finish_output2()`函数是IP层输出路径上的关键函数，它负责完成数据包发送前的最后准备工作。这个函数的核心任务是确保数据包具有正确的链路层头部信息，并通过邻居子系统查找或创建必要的ARP条目，核心流程如下。

- **按需创建**：只有在实际需要发送数据包时才创建ARP条目，避免不必要的内存占用。
- **缓存机制**：通过邻居表缓存已知的IP-MAC映射，提高后续数据包的发送效率。
- **错误处理**：对各种异常情况（如内存不足、ARP解析失败）进行妥善处理。

```C
// /Users/kangxiaoning/workspace/linux-3.10/net/ipv4/ip_output.c

static inline int ip_finish_output2(struct sk_buff *skb)
{
	struct dst_entry *dst = skb_dst(skb);
	// 从skb（socket buffer）中的dst_entry获取路由表项（rtable）
	struct rtable *rt = (struct rtable *)dst;
	struct net_device *dev = dst->dev;
	unsigned int hh_len = LL_RESERVED_SPACE(dev);
	struct neighbour *neigh;
	u32 nexthop;

    // 检查目标地址类型（RTN_MULTICAST或RTN_BROADCAST）
    // 如果是多播或广播地址，则更新相应的统计计数器
	if (rt->rt_type == RTN_MULTICAST) {
		IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTMCAST, skb->len);
	} else if (rt->rt_type == RTN_BROADCAST)
		IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTBCAST, skb->len);

	/* Be paranoid, rather than too clever. */
	// 如果skb头部没有足够空间存放链路层header并且设备有header_ops
	if (unlikely(skb_headroom(skb) < hh_len && dev->header_ops)) {
		struct sk_buff *skb2;

        // 尝试重新分配更大的header空间
		skb2 = skb_realloc_headroom(skb, LL_RESERVED_SPACE(dev));
		// 如果分配失败，则释放原始skb，返回-ENOMEM错误
		if (skb2 == NULL) {
			kfree_skb(skb);
			return -ENOMEM;
		}
		if (skb->sk)
			skb_set_owner_w(skb2, skb->sk);
		consume_skb(skb);
		skb = skb2;
	}

	rcu_read_lock_bh();
	// 通过rt_nexthop()计算下一跳地址
	nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
	
	// 查找与下一跳对应的邻居条目（ARP缓存项）
	neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
	// 如果没有找到，则尝试创建新的邻居条目
	if (unlikely(!neigh))
		neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);
	
	// 如果成功找到或创建了邻居条目，调用dst_neigh_output()函数进行实际的数据包发送操作
	if (!IS_ERR(neigh)) {
		int res = dst_neigh_output(dst, neigh, skb);

		rcu_read_unlock_bh();
		return res;
	}
	rcu_read_unlock_bh();

    // 如果成功找到或创建了邻居条目，调用dst_neigh_output()函数进行实际的数据包发送操作
	net_dbg_ratelimited("%s: No header cache and no neighbour!\n",
			    __func__);
	kfree_skb(skb);
	return -EINVAL;
}
```
{collapsible="true" collapsed-title="ip_finish_output2()" default-state="collapsed"}

前面提到将一个IP数据包发出去时，需要知道目标IP的MAC地址，此时会触发ARP请求。比如在容器网络中，经过策略路由后要发往Bridge的场景下，得先获取目标IP对应的MAC，然后再送到Bridge上，如果此时获取MAC失败，那Bridge就无法收到这个包，就会有丢包/重传的现象。

在Linux Kernel中，这个过程体现在`ip_finish_output2()`函数中，它的主要功能是根据路由表信息将IP数据包发送到正确的网络设备上，主要逻辑如下。
1. 从skb（socket buffer）中的dst_entry获取路由表项（rtable），并检查目标地址类型（RTN_MULTICAST或RTN_BROADCAST）。如果是多播或广播地址，则更新相应的统计计数器。
2. 检查skb头部预留的空间是否足够存放即将添加的链路层Header。如果不足够并且设备有header_ops，函数会尝试重新分配更大的Header空间。如果重新分配失败，则释放原始skb，并返回-ENOMEM错误。
3. 通过`rt_nexthop()`计算下一跳地址，并使用`__ipv4_neigh_lookup_noref()`查找与下一跳对应的邻居条目（ARP缓存项）。如果没有找到，则尝试创建新的邻居条目。
4. 如果成功找到或创建了邻居条目，调用`dst_neigh_output()`函数进行实际的数据包发送操作。此函数会利用链路层协议（如以太网）和邻居条目的信息来封装数据包，并将其发送到网络设备。
5. 发送完成后，释放读取锁（rcu_read_unlock_bh()）并返回结果。
6. 如果在查找或创建邻居条目时出错，打印错误日志，并释放skb，返回-EINVAL错误。

## 3. Neighbour条目创建及回收

### 3.1 强制GC

邻居表作为内核中的一个重要数据结构，需要在内存中维护大量的IP-MAC映射关系。为了防止邻居表无限制增长导致内存耗尽，Linux内核实现了一套垃圾回收机制。强制GC（Garbage Collection）是其中的一种机制，它在创建新条目时触发。

内核设置了三个阈值来控制邻居表的大小：
- **gc_thresh1**：邻居表条目数量的最小阈值，低于此值时不进行垃圾回收。
- **gc_thresh2**：软限制阈值，超过此值且距离上次清理超过5秒时会尝试垃圾回收。
- **gc_thresh3**：硬限制阈值，超过此值时必须进行垃圾回收，如果回收失败则拒绝创建新条目。

这种三级阈值设计既保证了系统的稳定性，又在正常情况下避免了频繁的垃圾回收操作，实现了性能和资源使用的平衡。

在创建neighbour条目时会判断是否要强制回收，代码如下。

```C
// /Users/kangxiaoning/workspace/linux-3.10/net/core/neighbour.c

static struct neighbour *neigh_alloc(struct neigh_table *tbl, struct net_device *dev)
{
	struct neighbour *n = NULL;
	unsigned long now = jiffies;
	int entries;

	entries = atomic_inc_return(&tbl->entries) - 1;
	if (entries >= tbl->gc_thresh3 ||
	    (entries >= tbl->gc_thresh2 &&
	     time_after(now, tbl->last_flush + 5 * HZ))) {
		if (!neigh_forced_gc(tbl) &&
		    entries >= tbl->gc_thresh3)
			goto out_entries;
	}

	n = kzalloc(tbl->entry_size + dev->neigh_priv_len, GFP_ATOMIC);
	if (!n)
		goto out_entries;

	skb_queue_head_init(&n->arp_queue);
	rwlock_init(&n->lock);
	seqlock_init(&n->ha_lock);
	n->updated	  = n->used = now;
	n->nud_state	  = NUD_NONE;
	n->output	  = neigh_blackhole;
	seqlock_init(&n->hh.hh_lock);
	n->parms	  = neigh_parms_clone(&tbl->parms);
	setup_timer(&n->timer, neigh_timer_handler, (unsigned long)n);

	NEIGH_CACHE_STAT_INC(tbl, allocs);
	n->tbl		  = tbl;
	atomic_set(&n->refcnt, 1);
	n->dead		  = 1;
out:
	return n;

out_entries:
	atomic_dec(&tbl->entries);
	goto out;
}
```
{collapsible="true" collapsed-title="neigh_alloc()" default-state="collapsed"}

Neighbour是个结构体，有个table来存储Neighbour条目，保存在内存中，因此它会有大小限制，在创建过程中会检查并做一些GC以确保不会无限制增长。具体逻辑实现在`neigh_alloc()`函数中。

1. 通过原子操作增加邻居表`struct neigh_table *tbl`中的条目计数，并获取当前条目数量。
2. 满足如下任一条件则尝试进行垃圾回收。
   - 条目数超过gc_thresh3
   - 条目数超过gc_thresh2但距离上一次刷新已经超过5秒
3. 如果垃圾回收失败，或者不需要垃圾回收但条目数超过最大条目数gc_thresh3，函数将直接返回空指针。
4. 使用`kzalloc()`为邻居条目分配内存，包括基本的邻居结构体大小和设备特定的私有数据长度。如果内存分配失败，也会返回空指针。
5. 成功分配内存后，函数初始化邻居条目的各个字段，关注下面几个，和GC有关。
    - 设置默认的output处理函数（这里是`neigh_blackhole()`，即丢弃报文）。
    - 初始化定时器（用于老化邻居条目）。
    - 更新统计信息并关联到对应的邻居表。
    - 设置引用计数。

### 3.2 周期性GC

除了在创建新条目时的强制GC，Linux内核还实现了周期性的垃圾回收机制。这种机制通过定时任务（delayed work）的方式实现，确保即使在网络流量较少的情况下，过期的邻居条目也能被及时清理。

周期性GC包括以下几个方面：
- **自适应的检查间隔**：根据`base_reachable_time`参数动态调整检查频率，通常为`base_reachable_time`的一半。
- **状态机管理**：基于邻居条目的状态（如NUD_FAILED、NUD_STALE等）来决定是否回收。
- **引用计数检查**：只有引用计数为1（即只有邻居表本身引用）的条目才会被回收。
- **老化时间控制**：通过`gc_staletime`参数控制条目的最大存活时间。

通过上面的设计既能及时清理无用条目，又避免了误删仍在使用的条目，保证了网络通信的稳定性。

```C
// /Users/kangxiaoning/workspace/linux-3.10/net/core/neighbour.c
static void neigh_periodic_work(struct work_struct *work)
{
	struct neigh_table *tbl = container_of(work, struct neigh_table, gc_work.work);
	struct neighbour *n;
	struct neighbour __rcu **np;
	unsigned int i;
	struct neigh_hash_table *nht;

	NEIGH_CACHE_STAT_INC(tbl, periodic_gc_runs);

	write_lock_bh(&tbl->lock);
	nht = rcu_dereference_protected(tbl->nht,
					lockdep_is_held(&tbl->lock));

	if (atomic_read(&tbl->entries) < tbl->gc_thresh1)
		goto out;

	/*
	 *	periodically recompute ReachableTime from random function
	 */

	if (time_after(jiffies, tbl->last_rand + 300 * HZ)) {
		struct neigh_parms *p;
		tbl->last_rand = jiffies;
		for (p = &tbl->parms; p; p = p->next)
			p->reachable_time =
				neigh_rand_reach_time(p->base_reachable_time);
	}

	for (i = 0 ; i < (1 << nht->hash_shift); i++) {
		np = &nht->hash_buckets[i];

		while ((n = rcu_dereference_protected(*np,
				lockdep_is_held(&tbl->lock))) != NULL) {
			unsigned int state;

			write_lock(&n->lock);

			state = n->nud_state;
			if (state & (NUD_PERMANENT | NUD_IN_TIMER)) {
				write_unlock(&n->lock);
				goto next_elt;
			}

			if (time_before(n->used, n->confirmed))
				n->used = n->confirmed;

			if (atomic_read(&n->refcnt) == 1 &&
			    (state == NUD_FAILED ||
			     time_after(jiffies, n->used + n->parms->gc_staletime))) {
				*np = n->next;
				n->dead = 1;
				write_unlock(&n->lock);
				neigh_cleanup_and_release(n);
				continue;
			}
			write_unlock(&n->lock);

next_elt:
			np = &n->next;
		}
		/*
		 * It's fine to release lock here, even if hash table
		 * grows while we are preempted.
		 */
		write_unlock_bh(&tbl->lock);
		cond_resched();
		write_lock_bh(&tbl->lock);
		nht = rcu_dereference_protected(tbl->nht,
						lockdep_is_held(&tbl->lock));
	}
out:
	/* Cycle through all hash buckets every base_reachable_time/2 ticks.
	 * ARP entry timeouts range from 1/2 base_reachable_time to 3/2
	 * base_reachable_time.
	 */
	schedule_delayed_work(&tbl->gc_work,
			      tbl->parms.base_reachable_time >> 1);
	write_unlock_bh(&tbl->lock);
}
```
{collapsible="true" collapsed-title="neigh_periodic_work()" default-state="collapsed"}

`neigh_periodic_work()`的核心逻辑如下：

1. 如果小于第一个阈值gc_thresh1，则直接跳出垃圾回收过程
2. 定期重新计算ReachableTime，如果距离上次随机计算超过5分钟（300秒），则根据base_reachable_time和随机函数更新所有`neigh_parms`结构体中的base_reachable_time，后续就会根据用户配置的时间定期GC。
3. 遍历neighbour hash table的hash_buckets，处理每个条目。
   - 如果该邻居条目为PERMANENT或正在进行定时器处理，则跳过。
   - 更新used字段以确保其不早于confirmed字段的时间。
   - 如果引用计数是否为1，并且状态为NUD_FAILED或者已经超过其对应的gc_staletime，则从链表中移除该条目，标记为dead，并调用`neigh_cleanup_and_release()`释放资源。

## 4. Neighbour条目GC逻辑

强制垃圾回收（`neigh_forced_gc`）是邻居表管理中的关键机制，它在系统资源紧张时被调用，用于快速释放不再使用的邻居条目。这个函数实现了一种简单而有效的回收策略：遍历整个哈希表，删除所有可以安全删除的条目。

核心实现逻辑如下：
- **引用计数保护**：只删除引用计数为1的条目，这意味着除了邻居表本身，没有其他地方正在使用该条目。
- **永久条目保护**：标记为NUD_PERMANENT的条目不会被删除，这些通常是管理员手动配置的静态ARP条目。
- **原子操作**：使用RCU（Read-Copy-Update）机制确保在多核环境下的并发安全。
- **快速响应**：通过一次遍历尽可能多地回收内存，减少系统压力。

```C
// /Users/kangxiaoning/workspace/linux-3.10/net/core/neighbour.c

static int neigh_forced_gc(struct neigh_table *tbl)
{
	int shrunk = 0;
	int i;
	struct neigh_hash_table *nht;

	NEIGH_CACHE_STAT_INC(tbl, forced_gc_runs);

	write_lock_bh(&tbl->lock);
	nht = rcu_dereference_protected(tbl->nht,
					lockdep_is_held(&tbl->lock));
	for (i = 0; i < (1 << nht->hash_shift); i++) {
		struct neighbour *n;
		struct neighbour __rcu **np;

		np = &nht->hash_buckets[i];
		while ((n = rcu_dereference_protected(*np,
					lockdep_is_held(&tbl->lock))) != NULL) {
			/* Neighbour record may be discarded if:
			 * - nobody refers to it.
			 * - it is not permanent
			 */
			write_lock(&n->lock);
			if (atomic_read(&n->refcnt) == 1 &&
			    !(n->nud_state & NUD_PERMANENT)) {
				rcu_assign_pointer(*np,
					rcu_dereference_protected(n->next,
						  lockdep_is_held(&tbl->lock)));
				n->dead = 1;
				shrunk	= 1;
				write_unlock(&n->lock);
				neigh_cleanup_and_release(n);
				continue;
			}
			write_unlock(&n->lock);
			np = &n->next;
		}
	}

	tbl->last_flush = jiffies;

	write_unlock_bh(&tbl->lock);

	return shrunk;
}
```
{collapsible="true" collapsed-title="neigh_forced_gc()" default-state="collapsed"}

根据源码可以看到删除条目要满足的条件：
- 引用计数是为1
- 不是PERMANENT的条目

## 5. 案例分析

一个因为arp cache满导致应用timeout的案例。

1. 现象

业务**高峰期**应用出现timeout。

2. 分析
  - 从业务高峰期才出现异常，怀疑是性能问题
  - 在Pod，Bridge，bond网卡等多个点抓包，对比正常和异常的数据包路径（见下图）
    - 正常情况下，在Bridge可以看到第6步的包
    - 异常情况下，在Bridge看不到第6步的包
  - 从内核处理逻辑进一步分析，怀疑与ARP Cache有关

![](arp-cache-full.svg)

3. 解决

将gc相关的3个参数调整为原来的2倍，持续观察未出现异常，问题解决。

```Shell
# 上限
cat /proc/sys/net/ipv4/neigh/default/gc_thresh3

# 当前数量
cat /proc/net/stat/arp_cache | head -2 | tail -1 | awk '{print "0x " $1}' | xargs printf "%d\n"
```
