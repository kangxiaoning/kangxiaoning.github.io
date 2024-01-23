# Linux Neighbor源码分析

## 1. 什么时候需要创建Neighbor条目

在Linux内核中，当网络数据包需要通过ARP（地址解析协议）或其他对应的协议（如IPv6下的NDP，Neighbor Discovery Protocol）来解析目标主机的硬件地址时，会创建neighbour条目(也就是ARP条目)，比如下列情况。

1. 发送数据包至本地网络上的未知MAC地址时，当IP层需要将一个IP数据包发送到同一链路上的一个目的IP地址，但不知道该目的IP对应的MAC地址时，内核会触发ARP请求以获取目标MAC地址，并且在这个过程中创建或更新neighbour条目。
2. 接收ARP/NDP响应时，当收到ARP应答或者NDP中的Neighbor Advertisement消息，其中包含了其他主机的IP与MAC地址对应关系时，内核会根据这些信息创建或更新neighbour条目。
3. 静态配置，可以通过命令行工具、配置文件等方式手动为特定的IP地址静态配置其对应的MAC地址，此时也会在neighbour表中创建相应的条目。
4. 路由更新时，当路由表发生变更，导致新的下一跳成为本地链路时，对于新的下一跳地址，如果之前没有对应的neighbour条目，则会触发ARP查询并创建新条目。
5. GARP(Gratuitous ARP)或者NS(Neighbor Solicitation)，接收到Gratuitous ARP或NDP的NS消息时，即使不是直接发给本机的，也可能用于更新或创建neighbour条目。

一句话总结，每当Linux需要或者从网络上获得到本地链路上的IP与MAC地址映射关系时，都会在neighbour cache（即ARP缓存或NDP缓存）中创建或更新相应条目。

## 2. IP层查找路由时更新neighbor条目

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
{collapsible="true" collapsed-title="neigh_alloc()" default-state="expanded"}

Neighbour是个结构体，有个table来存储Neighbour条目，保存在内存中，因此它会有大小限制，在创建过程中会检查并做一些GC以确保不会无限制增长。具体逻辑实现在`neigh_alloc()`函数中。

1. 通过原子操作增加邻居表（struct neigh_table *tbl）中的条目计数，并获取当前条目数量。
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

## 4. Neighbour条目GC逻辑

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
