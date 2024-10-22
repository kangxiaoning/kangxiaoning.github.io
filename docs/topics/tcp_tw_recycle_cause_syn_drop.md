# Tcp_tw_recycle导致SYN丢包

<show-structure depth="3"/>


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

收到`SYN`包后，内核会执行到如下函数，在这里做了**TIME_WAIT**状态下**SYN**包的逻辑处理。同一个客户端上次的连接还处于**TIME_WAIT**状态，当前时间和**TIME_WAIT**状态记录的时间差小于60秒，会被认为是不合理的SYN包，导致检查失败，SYN被drop，TCP连接建立失败。

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

设置`tcp_tw_recycle`的值为1。

