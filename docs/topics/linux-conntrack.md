# ConnTrack源码分析

<show-structure depth="3"/>

> 本文基于Linux kernel v3.10对ConnTrack工作原理进行分析。

## 1. Netfilter Packet Flow

<procedure>
<img src="netfilter-packet-flow.svg" alt="etcd service" thumbnail="true"/>
</procedure>

关注上图中conntrack的位置，包含了很多设计上的考虑。

1. 要跟踪**有效**连接，有效性要求数据包在**进入应用之前**或者**离开网卡之前**要**通过所有查检**。
2. 要**尽可能早**的跟踪连接，越早介入干扰越小。

要达成这些目的，必须确保数据包**通过所有检查**，并且**尽可能早**生成跟踪信息，因此这里涉及一个**开始生成**和**最终确认**的过程。
- **开始生成**
  - `PRE_ROUTING`是**接收方向**的数据包最先到达的地方，因此要在这个点生成跟踪信息。
  - `LOCAL_OUT`是**发送方向**的数据包最先开始的地方，因此要在这个点生成跟踪信息。
- **最终确认**
  - `LOCAL_IN`是数据包**通过所有检查**，**到达应用之前**的最后一个hook点，因此要在这个点确认。
  - `POST_ROUTING`是数据包**通过所有检查**，**离开主机之前**的最后一个hook点，因此要在这个点确认。

## 2. nf_conntrack_in()

```C
// /Users/kangxiaoning/workspace/linux-3.10/net/netfilter/nf_conntrack_core.c

// nf_conntrack_in是网络连接跟踪模块的核心函数，它处理进入的数据包并决定是否跟踪其连接状态。
unsigned int nf_conntrack_in(struct net *net, u_int8_t pf, unsigned int hooknum,
		struct sk_buff *skb)
{
    // 定义局部变量，包括待处理的nf_conn结构体指针(ct)、可能存在的模板结构体指针(tmpl)、连接跟踪信息枚举(ctinfo)等。
    struct nf_conn *ct, *tmpl = NULL;
    enum ip_conntrack_info ctinfo;
    // 获取三层协议（如IPv4/6）和四层协议（如TCP/UDP）处理模块。
    struct nf_conntrack_l3proto *l3proto;
    struct nf_conntrack_l4proto *l4proto;
    // 定义超时策略数组指针(timeouts)，用于存储不同连接阶段的超时时间。
    unsigned int *timeouts;
    // 数据偏移量(dataoff)和协议号(protonum)初始化为0。
    unsigned int dataoff;
    u_int8_t protonum;
    // set_reply标志，表示是否已接收到连接的回复数据包。
    int set_reply = 0;
    // 定义返回值ret，表示对数据包的处理结果。

    // 检查skb是否已关联nf_conn结构体，若已关联且不是模板，则忽略此数据包，并统计ignore计数器。
    if (skb->nfct) {
        tmpl = (struct nf_conn *)skb->nfct;
        if (!nf_ct_is_template(tmpl)) {
            NF_CT_STAT_INC_ATOMIC(net, ignore);
            return NF_ACCEPT;
        }
        skb->nfct = NULL; // 清除skb中关联的nf_conn指针，准备重新处理。
    }

    // 获取当前协议族对应的三层协议处理模块，并尝试获取四层协议相关信息。
    l3proto = __nf_ct_l3proto_find(pf);
    ret = l3proto->get_l4proto(skb, skb_network_offset(skb),
                               &dataoff, &protonum);
    // 如果获取失败（例如不支持的协议或错误发生），打印调试信息并更新统计计数器，然后根据错误码返回处理结果。
    if (ret <= 0) {
        pr_debug("not prepared to track yet or error occurred\n");
        NF_CT_STAT_INC_ATOMIC(net, error);
        NF_CT_STAT_INC_ATOMIC(net, invalid);
        ret = -ret;
        goto out;
    }

    // 获取四层协议处理模块。
    l4proto = __nf_ct_l4proto_find(pf, protonum);

    // 若四层协议处理模块提供了错误回调函数，调用该函数处理可能的错误或特殊类型的数据包。
    if (l4proto->error != NULL) {
        ret = l4proto->error(net, tmpl, skb, dataoff, &ctinfo,
                             pf, hooknum);
        // 如果处理返回负数或零，说明无法正常跟踪或需要丢弃数据包，更新统计计数器并返回处理结果。
        if (ret <= 0) {
            NF_CT_STAT_INC_ATOMIC(net, error);
            NF_CT_STAT_INC_ATOMIC(net, invalid);
            ret = -ret;
            goto out;
        }
        // 若ICMP[v6]等协议处理模块已为数据包分配了nf_conn，则跳转到out标签进行后续处理。
        if (skb->nfct)
            goto out;
    }

    // 尝试解析或创建一个正常的nf_conn结构体来跟踪此次连接交互。
    ct = resolve_normal_ct(net, tmpl, skb, dataoff, pf, protonum,
                           l3proto, l4proto, &set_reply, &ctinfo);
    // 如果无法创建有效的nf_conn结构体，说明数据包不属于任何已知连接，更新invalid计数器并接受数据包。
    if (!ct) {
        NF_CT_STAT_INC_ATOMIC(net, invalid);
        ret = NF_ACCEPT;
        goto out;
    }

    // 如果resolve_normal_ct返回的是错误（通过IS_ERR宏判断），则表明系统资源紧张无法处理，丢弃数据包并更新drop计数器。
    if (IS_ERR(ct)) {
        NF_CT_STAT_INC_ATOMIC(net, drop);
        ret = NF_DROP;
        goto out;
    }

    // 确保skb已关联nf_conn结构体。
    NF_CT_ASSERT(skb->nfct);

    // 根据协议及连接状态查找并获取适用于当前连接的超时策略数组。
    timeouts = nf_ct_timeout_lookup(net, ct, l4proto);

    // 调用四层协议处理模块的packet函数深度处理数据包内容，更新连接状态等信息。
    ret = l4proto->packet(ct, skb, dataoff, ctinfo, pf, hooknum, timeouts);
    // 如果处理失败，释放nf_conn引用，清空skb中的nf_conn指针，并更新统计计数器，最后返回处理结果。
    if (ret <= 0) {
        pr_debug("nf_conntrack_in: Can't track with proto module\n");
        nf_conntrack_put(skb->nfct);
        skb->nfct = NULL;
        NF_CT_STAT_INC_ATOMIC(net, invalid);
        if (ret == -NF_DROP)
            NF_CT_STAT_INC_ATOMIC(net, drop);
        ret = -ret;
        goto out;
    }

    // 如果这是连接的回复数据包并且之前未标记过，则更新连接状态并触发事件通知。
    if (set_reply && !test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct->status))
        nf_conntrack_event_cache(IPCT_REPLY, ct);

out:
    // 清理模板引用。如果需要重复处理该数据包（nf_repeat），则重新将tmpl赋值给skb；否则释放tmpl引用。
    if (tmpl) {
        if (ret == NF_REPEAT)
            skb->nfct = (struct nf_conntrack *)tmpl;
        else
            nf_ct_put(tmpl);
    }

    // 返回处理结果。
    return ret;
}
```
{collapsible="true" collapsed-title="nf_conntrack_in()" default-state="collapsed"}

`nf_conntrack_in()`的主要作用是对进入的数据包进行连接跟踪处理，以实现对网络连接状态的维护和控制。

## 3. resolve_normal_ct()

```C
// /Users/kangxiaoning/workspace/linux-3.10/net/netfilter/nf_conntrack_core.c

// resolve_normal_ct函数用于根据给定的数据包信息查找或创建一个新的nf_conn结构体，用于连接跟踪。若成功，返回nf_conn指针，并将连接跟踪信息设置到skb和ctinfo中。

static inline struct nf_conn * // 返回类型为nf_conn结构体指针
resolve_normal_ct(struct net *net, // 当前网络子系统结构体
		  struct nf_conn *tmpl, // 可选的模板连接跟踪结构体（如SNAT/DNAT）
		  struct sk_buff *skb, // 数据包结构体
		  unsigned int dataoff, // 四层协议头部偏移量
		  u_int16_t l3num, // 三层协议号（例如IPPROTO_IP、IPPROTO_IPV6等）
		  u_int8_t protonum, // 四层协议号（例如TCP、UDP等）
		  struct nf_conntrack_l3proto *l3proto, // 三层协议处理模块
		  struct nf_conntrack_l4proto *l4proto, // 四层协议处理模块
		  int *set_reply, // 指示是否是回复方向数据包的标志
		  enum ip_conntrack_info *ctinfo) // 连接跟踪信息枚举值

{
    // 创建一个nf_conntrack_tuple结构体来存储当前数据包的关键信息。
    struct nf_conntrack_tuple tuple;
    // 定义指向nf_conntrack_tuple_hash结构体的指针，该结构体包含连接跟踪元组及nf_conn引用。
    struct nf_conntrack_tuple_hash *h;
    // 定义nf_conn结构体指针。
    struct nf_conn *ct;

    // 获取用于确定区域的zone值，如果tmpl存在，则使用tmpl关联的区域，否则使用默认区域。
    u16 zone = tmpl ? nf_ct_zone(tmpl) : NF_CT_DEFAULT_ZONE;
    // 计算连接跟踪元组的哈希值。
    u32 hash;

    // 根据skb中的数据获取并填充nf_conntrack_tuple结构体，若失败则打印错误信息并返回NULL。
    if (!nf_ct_get_tuple(skb, skb_network_offset(skb),
			     dataoff, l3num, protonum, &tuple, l3proto,
			     l4proto)) {
        pr_debug("resolve_normal_ct: Can't get tuple\n");
        return NULL;
    }

    // 查找与当前元组匹配的nf_conntrack_tuple_hash结构体。
    hash = hash_conntrack_raw(&tuple, zone);
    h = __nf_conntrack_find_get(net, zone, &tuple, hash);

    // 若未找到匹配项，则尝试初始化新的连接跟踪条目。
    if (!h) {
        h = init_conntrack(net, tmpl, &tuple, l3proto, l4proto,
                           skb, dataoff, hash);
        // 初始化失败，返回NULL。
        if (!h)
            return NULL;
        // 如果初始化过程中出现错误，转换为错误指针并返回。
        if (IS_ERR(h))
            return (void *)h;
    }
    // 将找到或新建的nf_conntrack_tuple_hash结构体转换为nf_conn结构体指针。
    ct = nf_ct_tuplehash_to_ctrack(h);

    // 已经找到了对应的连接跟踪条目，对数据包的方向进行判断：
    if (NF_CT_DIRECTION(h) == IP_CT_DIR_REPLY) {
        // 若为回复方向数据包，则设置连接跟踪信息为已建立的回复状态。
        *ctinfo = IP_CT_ESTABLISHED_REPLY;
        // 请求在后续处理中设置回复位（如果此数据包有效）。
        *set_reply = 1;
    } else {
        // 若双向通信已完成（即已收到回复），始终认为是已建立状态。
        if (test_bit(IPS_SEEN_REPLY_BIT, &ct->status)) {
            pr_debug("nf_conntrack_in: normal packet for %p\n", ct);
            *ctinfo = IP_CT_ESTABLISHED;
        } else if (test_bit(IPS_EXPECTED_BIT, &ct->status)) {
            // 若该数据包属于相关联的连接（如FTP辅助数据流），则设置为相关状态。
            pr_debug("nf_conntrack_in: related packet for %p\n", ct);
            *ctinfo = IP_CT_RELATED;
        } else {
            // 若为新连接的第一个数据包，则设置为新建状态。
            pr_debug("nf_conntrack_in: new packet for %p\n", ct);
            *ctinfo = IP_CT_NEW;
        }
        // 非回复方向数据包，默认设置set_reply为0。
        *set_reply = 0;
    }

    // 将nf_conn结构体地址赋值给skb的nfct字段，便于后续引用。
    skb->nfct = &ct->ct_general;
    // 设置skb的nfctinfo字段，记录当前数据包在连接中的状态。
    skb->nfctinfo = *ctinfo;

    // 最后返回nf_conn结构体指针。
    return ct;
}
```
{collapsible="true" collapsed-title="resolve_normal_ct()" default-state="collapsed"}

## 4. UDP的conntrack

在Linux内核的Netfilter连接跟踪子系统中，`nf_conntrack_tuple`结构体用于存储网络数据包的关键信息以唯一标识一个网络连接的方向。对于UDP协议，`nf_conntrack_tuple`结构通常包含以下信息：

- **源IP地址**（sip）：发送方IP地址。
- **目标IP地址**（dip）：接收方IP地址。
- **源端口号**（sport）：发送方使用的UDP端口号。
- **目的端口号**（dport）：接收方监听的UDP端口号。
- **协议号**（proto）：在网络层通常是IPPROTO_UDP。

根据上术信息，得出如下结论
1. 如果上述tuple的所有信息都相同，那么会命中同一个conntrack记录。
 
## 5. 案例分析

在kubernetes环境中，使用UDP协议解析域名，`coreDNS`异常重启，出现如下情况。
1. 应用POD的`resolv.conf`文件中的nameserver配置相同的VIP，因此`nf_conntrack_tuple`中的`dip/dport/proto`都相同。
2. 如果应用IP和端口号不变，也就是`nf_conntrack_tuple`中的`sip/sport`也相同，那发送方向的`nf_conntrack_tuple`的所有元素都相同了。
3. 此时`coreDNS`重启。
4. 如果上述应用持续向`coreDNS`发送请求，可能会命中kernel bug，导致无效conntrack记录未删除。
5. 因为发送方向上`nf_conntrack_tuple`的所有元素都相同，新的DNS解析数据包，命中无效的conntrack记录。

此时会导致什么问题？

