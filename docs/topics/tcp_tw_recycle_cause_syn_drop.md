# Tcp_tw_recycle导致SYN丢包

<show-structure depth="3"/>

TCP 协议中的 TIME_WAIT 状态是保证网络连接可靠性的重要机制，它能防止历史连接的数据包被新连接误接收。然而，在 NAT 环境下，TIME_WAIT 状态的复用机制可能导致意外的连接问题。本文结合实际案例深入分析 TCP TIME_WAIT 状态的工作原理，详细解释 tcp_tw_recycle 参数在 NAT 场景下引发 SYN 丢包的技术原因，并提供相应的解决方案。

为了防止**历史连接**中的数据被**相同四元组的新连接**接收，TCP设计了**TIME_WAIT**状态， 该状态会持续**2MSL**(Maximum Segment Lifetime)时⻓，这个时间足以让两个方向上的数据包都被丢弃，使得原来连接的数据包在网络中都自然消失，再收到的数据包一定都是**新连接**所产生的。

为了防止序列号回绕，内核有**PAWS**(Protect Against Wrapped Sequences)检查，在**NAT**场景下复用**TIME_WAIT**的逻辑不可靠，因此要关闭`tcp_tw_recycle`，否则会出现**SYN丢包**现象。

> 本文参考代码版本为：linux-3.10.0-1160。
>
{style="note"}

## 1. 问题现象

1. Client通过F5访问Server，经常出现502报错，在Server抓包，发现F5发送SYN后Server没有回复SYN+ACK，随后F5发送RST终止了连接。
2. `netstat -s`能观察到如下信息的数量在变化。
   - `xxx passive connections rejected because of time stamp`
   - `xxx SYNs to LISTEN sockets dropped`

## 2. 原因分析

### 2.1 TCP 连接请求处理核心机制

收到`SYN`包后，内核会执行到如下函数，在这里做了**TIME_WAIT**状态下**SYN**包的逻辑处理。同一个客户端上次的连接还处于**TIME_WAIT**状态，当前时间和**TIME_WAIT**状态记录的时间差小于60秒，会被认为是不合理的SYN包，导致检查失败，SYN被drop，TCP连接建立失败。

`tcp_conn_request()`函数是 TCP 服务端处理连接请求的核心入口点。该函数实现了完整的 SYN 包处理流程，包括 SYN Cookie 机制、TIME_WAIT 状态复用检查、PAWS (Protect Against Wrapped Sequences) 验证等。在启用 tcp_tw_recycle 的情况下，内核会对来自相同 IP 的连接请求进行时间戳验证，以确保新连接不会与处于 TIME_WAIT 状态的历史连接产生冲突。这种机制的设计初衷是优化 TIME_WAIT 状态的处理，但在 NAT 环境下可能导致误判。


```C
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	struct tcp_options_received tmp_opt;
	struct request_sock *req;
	struct tcp_sock *tp = tcp_sk(sk);
	struct dst_entry *dst = NULL;
	__u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn;
	bool want_cookie = false, fastopen;
	struct flowi fl;
	struct tcp_fastopen_cookie foc = { .len = -1 };
	int err;


	/* TW buckets are converted to open requests without
	 * limitations, they conserve resources and peer is
	 * evidently real one.
	 */
	if ((sysctl_tcp_syncookies == 2 ||
	     inet_csk_reqsk_queue_is_full(sk)) && !isn) {
		want_cookie = tcp_syn_flood_action(sk, skb, rsk_ops->slab_name);
		if (!want_cookie)
			goto drop;
	}


	/* Accept backlog is full. If we have already queued enough
	 * of warm entries in syn queue, drop request. It is better than
	 * clogging syn queue with openreqs with exponentially increasing
	 * timeout.
	 */
	if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) {
		NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	}

	req = inet_reqsk_alloc(rsk_ops);
	if (!req)
		goto drop;

	inet_rsk(req)->ireq_family = sk->sk_family;

	tcp_rsk(req)->af_specific = af_ops;

	tcp_clear_options(&tmp_opt);
	tmp_opt.mss_clamp = af_ops->mss_clamp;
	tmp_opt.user_mss  = tp->rx_opt.user_mss;
	tcp_parse_options(skb, &tmp_opt, 0, want_cookie ? NULL : &foc);

	if (want_cookie && !tmp_opt.saw_tstamp)
		tcp_clear_options(&tmp_opt);

	tmp_opt.tstamp_ok = tmp_opt.saw_tstamp;
	tcp_openreq_init(req, &tmp_opt, skb);

	af_ops->init_req(req, sk, skb);

	if (security_inet_conn_request(sk, skb, req))
		goto drop_and_free;

	if (!want_cookie && !isn) {
		/* VJ's idea. We save last timestamp seen
		 * from the destination in peer table, when entering
		 * state TIME-WAIT, and check against it before
		 * accepting new connection request.
		 *
		 * If "isn" is not zero, this request hit alive
		 * timewait bucket, so that all the necessary checks
		 * are made in the function processing timewait state.
		 */
		// 启用 tcp_tw_recycle
		if (tcp_death_row.sysctl_tw_recycle) {
			bool strict;

			dst = af_ops->route_req(sk, &fl, req, &strict);

            // 验证客户端
			if (dst && strict &&
			    // tcp_peer_is_proven返回false
			    !tcp_peer_is_proven(req, dst, true,
						tmp_opt.saw_tstamp)) {
				// 上述验证通过就增加统计信息
				// netstat -s | grep "passive connections rejected"
				// 能观察到数量增加
				NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_PAWSPASSIVEREJECTED);
				goto drop_and_release;
			}
		}
		/* Kill the following clause, if you dislike this way. */
		else if (!sysctl_tcp_syncookies &&
			 (sysctl_max_syn_backlog - inet_csk_reqsk_queue_len(sk) <
			  (sysctl_max_syn_backlog >> 2)) &&
			 !tcp_peer_is_proven(req, dst, false,
					     tmp_opt.saw_tstamp)) {
			/* Without syncookies last quarter of
			 * backlog is filled with destinations,
			 * proven to be alive.
			 * It means that we continue to communicate
			 * to destinations, already remembered
			 * to the moment of synflood.
			 */
			pr_drop_req(req, ntohs(tcp_hdr(skb)->source),
				    rsk_ops->family);
			goto drop_and_release;
		}

		isn = af_ops->init_seq(skb);
	}
	if (!dst) {
		dst = af_ops->route_req(sk, &fl, req, NULL);
		if (!dst)
			goto drop_and_free;
	}

	tcp_ecn_create_request(req, skb, sk, dst);

	if (want_cookie) {
		isn = cookie_init_sequence(af_ops, sk, skb, &req->mss);
		req->cookie_ts = tmp_opt.tstamp_ok;
		if (!tmp_opt.tstamp_ok)
			inet_rsk(req)->ecn_ok = 0;
	}

	tcp_rsk(req)->snt_isn = isn;
	tcp_openreq_init_rwin(req, sk, dst);
	fastopen = !want_cookie &&
		   tcp_try_fastopen(sk, skb, req, &foc, dst);
	err = af_ops->send_synack(sk, dst, &fl, req,
				  skb_get_queue_mapping(skb), &foc);
	if (!fastopen) {
		if (err || want_cookie)
			goto drop_and_free;

		tcp_rsk(req)->listener = NULL;
		af_ops->queue_hash_add(sk, req, TCP_TIMEOUT_INIT);
	}

	return 0;

drop_and_release:
	dst_release(dst);
drop_and_free:
	reqsk_free(req);
drop:
	NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENDROPS);
	return 0;
}
```
{collapsible="true" collapsed-title="tcp_conn_request" default-state="collapsed"}

### 2.2 新版本内核的改进机制

在 4.19.90 版本的内核中，tcp_tw_recycle 相关逻辑已被完全移除。新版本的实现更加简洁和安全，移除了可能在 NAT 环境下引发问题的时间戳验证机制。取而代之的是更可靠的连接验证策略，只在非 SYN Cookie 模式下对已验证的对等端进行检查，避免了 NAT 环境下的误判问题。

如下**4.19.90**版本的内核代码已经没有`tcp_tw_cycle`相关逻辑，据查在**4.12**版本后取消了。
```C
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	struct tcp_fastopen_cookie foc = { .len = -1 };
	__u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn;
	struct tcp_options_received tmp_opt;
	struct tcp_sock *tp = tcp_sk(sk);
	struct net *net = sock_net(sk);
	struct sock *fastopen_sk = NULL;
	struct request_sock *req;
	bool want_cookie = false;
	struct dst_entry *dst;
	struct flowi fl;

	/* TW buckets are converted to open requests without
	 * limitations, they conserve resources and peer is
	 * evidently real one.
	 */
	if ((net->ipv4.sysctl_tcp_syncookies == 2 ||
	     inet_csk_reqsk_queue_is_full(sk)) && !isn) {
		want_cookie = tcp_syn_flood_action(sk, skb, rsk_ops->slab_name);
		if (!want_cookie)
			goto drop;
	}

	if (sk_acceptq_is_full(sk)) {
		NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	}

	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);
	if (!req)
		goto drop;

	tcp_rsk(req)->af_specific = af_ops;
	tcp_rsk(req)->ts_off = 0;

	tcp_clear_options(&tmp_opt);
	tmp_opt.mss_clamp = af_ops->mss_clamp;
	tmp_opt.user_mss  = tp->rx_opt.user_mss;
	tcp_parse_options(sock_net(sk), skb, &tmp_opt, 0,
			  want_cookie ? NULL : &foc);

	if (want_cookie && !tmp_opt.saw_tstamp)
		tcp_clear_options(&tmp_opt);

	if (IS_ENABLED(CONFIG_SMC) && want_cookie)
		tmp_opt.smc_ok = 0;

	tmp_opt.tstamp_ok = tmp_opt.saw_tstamp;
	tcp_openreq_init(req, &tmp_opt, skb, sk);
	inet_rsk(req)->no_srccheck = inet_sk(sk)->transparent;

	/* Note: tcp_v6_init_req() might override ir_iif for link locals */
	inet_rsk(req)->ir_iif = inet_request_bound_dev_if(sk, skb);

	af_ops->init_req(req, sk, skb);

	if (security_inet_conn_request(sk, skb, req))
		goto drop_and_free;

	if (tmp_opt.tstamp_ok)
		tcp_rsk(req)->ts_off = af_ops->init_ts_off(net, skb);

	dst = af_ops->route_req(sk, &fl, req);
	if (!dst)
		goto drop_and_free;

	if (!want_cookie && !isn) {
		/* Kill the following clause, if you dislike this way. */
		if (!net->ipv4.sysctl_tcp_syncookies &&
		    (net->ipv4.sysctl_max_syn_backlog - inet_csk_reqsk_queue_len(sk) <
		     (net->ipv4.sysctl_max_syn_backlog >> 2)) &&
		    !tcp_peer_is_proven(req, dst)) {
			/* Without syncookies last quarter of
			 * backlog is filled with destinations,
			 * proven to be alive.
			 * It means that we continue to communicate
			 * to destinations, already remembered
			 * to the moment of synflood.
			 */
			pr_drop_req(req, ntohs(tcp_hdr(skb)->source),
				    rsk_ops->family);
			goto drop_and_release;
		}

		isn = af_ops->init_seq(skb);
	}

	tcp_ecn_create_request(req, skb, sk, dst);

	if (want_cookie) {
		isn = cookie_init_sequence(af_ops, sk, skb, &req->mss);
		req->cookie_ts = tmp_opt.tstamp_ok;
		if (!tmp_opt.tstamp_ok)
			inet_rsk(req)->ecn_ok = 0;
	}

	tcp_rsk(req)->snt_isn = isn;
	tcp_rsk(req)->txhash = net_tx_rndhash();
	tcp_openreq_init_rwin(req, sk, dst);
	sk_rx_queue_set(req_to_sk(req), skb);
	if (!want_cookie) {
		tcp_reqsk_record_syn(sk, req, skb);
		fastopen_sk = tcp_try_fastopen(sk, skb, req, &foc, dst);
	}
	if (fastopen_sk) {
		af_ops->send_synack(fastopen_sk, dst, &fl, req,
				    &foc, TCP_SYNACK_FASTOPEN);
		/* Add the child socket directly into the accept queue */
		if (!inet_csk_reqsk_queue_add(sk, req, fastopen_sk)) {
			reqsk_fastopen_remove(fastopen_sk, req, false);
			bh_unlock_sock(fastopen_sk);
			sock_put(fastopen_sk);
			reqsk_put(req);
			goto drop;
		}
		sk->sk_data_ready(sk);
		bh_unlock_sock(fastopen_sk);
		sock_put(fastopen_sk);
	} else {
		tcp_rsk(req)->tfo_listener = false;
		if (!want_cookie)
			inet_csk_reqsk_queue_hash_add(sk, req,
				tcp_timeout_init((struct sock *)req));
		af_ops->send_synack(sk, dst, &fl, req, &foc,
				    !want_cookie ? TCP_SYNACK_NORMAL :
						   TCP_SYNACK_COOKIE);
		if (want_cookie) {
			reqsk_free(req);
			return 0;
		}
	}
	reqsk_put(req);
	return 0;

drop_and_release:
	dst_release(dst);
drop_and_free:
	reqsk_free(req);
drop:
	tcp_listendrop(sk);
	return 0;
}
```
{collapsible="true" collapsed-title="tcp_conn_request" default-state="collapsed"}

### 2.3 TIME_WAIT 状态时间参数定义

如下这些常量定义了 TCP TIME_WAIT 状态的关键时间参数。TCP_TIMEWAIT_LEN (60秒) 定义了 TIME_WAIT 状态的持续时间。TCP_PAWS_MSL (60秒) 是 PAWS 机制的核心参数，它定义了每个主机时间戳的有效期。在这个时间窗口内，来自同一主机的连接请求会受到严格的时间戳检查，以防止序列号回绕攻击。TCP_PAWS_WINDOW (1秒) 定义了时间戳回放检测的窗口大小。

```C
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
				  * state, about 60 seconds	*/
#define TCP_FIN_TIMEOUT	TCP_TIMEWAIT_LEN
                                 /* BSD style FIN_WAIT2 deadlock breaker.
				  * It used to be 3min, new value is 60sec,
				  * to combine FIN-WAIT-2 timeout with
				  * TIME-WAIT timer.
				  */

// 每个主机的socket会记录timestamps，在60s后被视为无效，这样该host新建立的连接
// 就可以可靠的复用TIME_WAIT状态的socket
// 如果在60s内收到同一个host的SYN包，无法判断是上一个连接的包还是新连接的包，因此
// 为了可靠处理TIME_WAIT状态复用场景，协议会drop这个SYN包
#define TCP_PAWS_MSL	60		/* Per-host timestamps are invalidated
					 * after this time. It should be equal
					 * (or greater than) TCP_TIMEWAIT_LEN
					 * to provide reliability equal to one
					 * provided by timewait state.
					 */
#define TCP_PAWS_WINDOW	1		/* Replay window for per-host
					 * timestamps. It must be less than
					 * minimal timewait lifetime.
					 */
```

### 2.4 对等端验证机制

`tcp_peer_is_proven()`函数实现了基于时间戳的对等端验证机制。该函数的核心逻辑是检查来自相同 IP 地址的连接请求是否可信。当启用 PAWS 检查时，函数会比较当前时间与记录的时间戳，如果时间差小于 TCP_PAWS_MSL (60秒)，且时间戳不符合预期，则认为这是一个可疑的连接请求。在 NAT 环境下，多个真实客户端会共享同一个公网 IP，导致时间戳检查失败，从而引发 SYN 包被错误丢弃的问题。这正是 NAT 环境下 tcp_tw_recycle 不可靠的根本原因。

```C
bool tcp_peer_is_proven(struct request_sock *req, struct dst_entry *dst,
			bool paws_check, bool timestamps)
{
	struct tcp_metrics_block *tm;
	bool ret;

	if (!dst)
		return false;

	rcu_read_lock();
	tm = __tcp_get_metrics_req(req, dst);
	if (paws_check) {
		if (tm &&
		    // 见上述TCP_PAWS_MSL注释，这里的判断是同一个IP，而不是IP+PORT
		    // 客户端通过F5 SNAT IP访问Server的场景，Server上存在TIME_WAIT状态的连接
		    // 当前时间减去TIME_WAIT连接的timestamp，如果小于60s，本次F5 SNAT IP发送的SYN包会被丢弃
		    // tcpdump可以观察到F5发送SYN，并且重传，但是Server不响应，直接drop
		    (u32)get_seconds() - tm->tcpm_ts_stamp < TCP_PAWS_MSL &&
		    ((s32)(tm->tcpm_ts - req->ts_recent) > TCP_PAWS_WINDOW ||
		     !timestamps))
			ret = false;
		else
			ret = true;
	} else {
		if (tm && tcp_metric_get(tm, TCP_METRIC_RTT) && tm->tcpm_ts_stamp)
			ret = true;
		else
			ret = false;
	}
	rcu_read_unlock();

	return ret;
}
```

## 3. 解决方案

### 3.1 立即解决方案

设置`tcp_tw_recycle`的值为**0**（关闭）。

```bash
# 临时设置
echo 0 > /proc/sys/net/ipv4/tcp_tw_recycle

# 永久设置
echo 'net.ipv4.tcp_tw_recycle = 0' >> /etc/sysctl.conf
sysctl -p
```

### 3.2 长期优化建议

1. **升级内核版本**: 建议升级到 4.12 以上版本，新版本内核已移除 tcp_tw_recycle 参数，从根本上解决了该问题。

2. **调整 TIME_WAIT 相关参数**:
   ```bash
   # 启用 TIME_WAIT socket 复用
   net.ipv4.tcp_tw_reuse = 1
   
   # 调整 TIME_WAIT 超时时间
   net.ipv4.tcp_fin_timeout = 30
   ```
> **Best practices and recommendations**
> 
> **Never enable tcp_tw_recycle** - this recommendation is now automatically enforced since the parameter no longer exists in modern kernels. Remove any references to tcp_tw_recycle from configuration files to avoid error messages.
> 
> **For tcp_tw_reuse, use value 2 (loopback-only) as the safest option** for production systems. This provides benefits for local inter-service communication while avoiding risks associated with external connections. Only consider value 1 in controlled environments without NAT devices.

3. **应用层优化**: 使用连接池技术减少连接创建和销毁的频率，降低 TIME_WAIT 状态的影响。

## 4. 技术影响分析

### 4.1 NAT 环境下的问题根源

在 NAT 环境中，多个内网客户端通过同一个公网 IP 访问服务器。当启用 tcp_tw_recycle 时，服务器端的时间戳检查机制会将来自同一公网 IP 的所有连接视为同一个客户端，导致：

- 时间戳不连续的 SYN 包被误判为重复包
- 正常的连接请求被错误丢弃
- 业务出现间歇性连接失败

### 4.2 性能与安全的权衡

tcp_tw_recycle 机制的设计初衷是：
- **优势**: 加速 TIME_WAIT 状态的回收，提高端口复用效率
- **劣势**: 在 NAT 环境下可靠性不足，容易引发连接问题
- **演进**: 新版本内核通过更安全的机制替代了这一功能

## 总结

TCP TIME_WAIT 状态和 tcp_tw_recycle 机制体现了网络协议设计中性能与可靠性的权衡考虑。

**核心机制**：
- TIME_WAIT 状态通过 2MSL 等待时间确保连接的可靠关闭
- PAWS 机制防止序列号回绕攻击，保护连接安全
- tcp_tw_recycle 通过时间戳验证加速 TIME_WAIT 状态复用

**问题根源**：
- 在 NAT 环境下，多客户端共享 IP 导致时间戳检查失效
- 基于 IP 而非四元组的验证逻辑存在设计缺陷
- 60秒的时间窗口在高并发场景下容易触发误判

**解决方案**：
- 短期：关闭 tcp_tw_recycle 参数
- 长期：升级到新版本内核，使用更可靠的替代机制
- 应用层：采用连接池等技术减少连接创建频率

**技术演进**：
- 在 4.12 版本后完全移除了 tcp_tw_recycle，转向更安全可靠的实现方式。
