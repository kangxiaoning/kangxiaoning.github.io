# 探索Linux内核网络协议栈

<show-structure depth="3"/>

关于Linux内核网络源码的学习记录。

## 1. 注册`Protocol Family`

Linux内核在初始化阶段会执行`inet_init`函数，该函数注册了`AF_INET/PF_INET`这个Protocol Family以及该Family对应的协议栈(`TCP、UDP、ICMP、RAW`)，相关代码在`net/ipv4/afinet.c`。

1. 全局数组`net_families[]`用于保存已注册的`Protocol Family`指针。
```C
static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;
```

2. `inet_init`中通过如下代码将`inet_family_ops`注册到`net_families[]`中。
```C
	(void)sock_register(&inet_family_ops);
```

3. `inet_family_ops`定义及`PF_INET`定义如下。
```C
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};
```

```C
// /home/kangxiaoning/workspace/linux-3.10.0-1160/include/linux/socket.h

#define PF_INET		AF_INET

#define AF_INET		2	/* Internet IP Protocol 	*/
```

`inet_init`执行后，`net_families[PF_INET]`就指向了`inet_family_ops`结构体。

## 2. 创建`Socket`

### 2.1 `Family`匹配过程

在C语言中通过`socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)`创建一个socket。

1. 调用`socket`syscall。

```C
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
```

2. 调用`sock_create`创建名为`sock`的`struct socket`对象。
```C
	retval = sock_create(family, type, protocol, &sock);
```

3. 在`__sock_create`函数中通过`net_families[family]`获取`Protocol Family`，根据传入的**AF_INET**得到`inet_family_ops`。
```C
	pf = rcu_dereference(net_families[family]);
```

4. 调用`pf->create`创建`socket`，即`inet_family_ops`中的`inet_create`函数。
```C
	err = pf->create(net, sock, protocol, kern);
```

至此，确定了family为`AF_INET`的socket应该调用`inet_create`函数创建。

### 2.2 `Protocol`匹配过程

1. 全局数组`inetsw`维护socket type和protocol的关系，通过socket type(`SOCK_STREAM, SOCK_DGRAM, SOCK_RAW`)索引protocol(`TCP, UDP, ICMP, RAW`)。
```C
/* The inetsw table contains everything that inet_create needs to
 * build a new socket.
 */
static struct list_head inetsw[SOCK_MAX];
```

2. 静态数组`inetsw_array[]`包含默认protocol信息。
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
		.ops =        &inet_dgram_ops,
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

3. 在`inet_init`中将`inetsw_array[]`中的信息注册到`inetsw[]`中。

```C
	/* Register the socket-side information for inet_create. */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);
```

4. 在`inet_create`中利用注册的信息完成socket创建。

- 通过`inetsw[SOCK_STREAM]`找到`IPPROTO_TCP`的协议结构体。
```C
	{
		.type =       SOCK_STREAM,
		.protocol =   IPPROTO_TCP,
		.prot =       &tcp_prot,
		.ops =        &inet_stream_ops,
		.flags =      INET_PROTOSW_PERMANENT |
			      INET_PROTOSW_ICSK,
	},
```
- 完成`sock`及`sk`的初始化，`sock->ops`赋值为`&inet_stream_ops`，`sk->sk_prot`赋值为`&tcp_prot`。
```C
    // `sock->ops`赋值为`&inet_stream_ops`
	sock->ops = answer->ops;
	answer_prot = answer->prot;
	answer_flags = answer->flags;
	
	// `sk->sk_prot`赋值为`&tcp_prot`
	sk = sk_alloc(net, pf_inet, gfp_kernel, answer_prot);
```

至此，`socket`创建完成，协议栈层级结构中不同的层都完成了创始化，`socket`层与`protocol`层的关系如下。

```bash
User Space Application
        ↓
   socket(2) syscall
        ↓
┌────────────────────────────────┐
│   proto_ops (inet_stream_ops)  │ ← BSD Socket Layer (Generic)
│   - bind(), connect()          │
│   - send(), recv()             │
└────────────────────────────────┘
        ↓
┌─────────────────────────────┐
│   proto (tcp_prot)          │ ← Protocol-Specific Layer
│   - TCP state machine       │
│   - Congestion control      │
│   - Segment handling        │
└─────────────────────────────┘
        ↓
    IP Layer & Below
```

## 3. 操作集

如下列举了常见协议TCP/UDP的Socket操作集与Protocol操作集。

### 3.1 `SOCK_STREAM`操作集

`inet_stream_ops`定义了`socket`层中`SOCK_STREAM`类型的操作集。
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
	.mmap		   = sock_no_mmap,
	.sendpage	   = inet_sendpage,
	.splice_read	   = tcp_splice_read,
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_sock_common_setsockopt,
	.compat_getsockopt = compat_sock_common_getsockopt,
	.compat_ioctl	   = inet_compat_ioctl,
#endif
};
```

### 3.2 `TCP`操作集

`tcp_prot`定义了`TCP`协议的操作集，是`SOCK_STREAM`类型的一种具体实现。
```C
struct proto tcp_prot = {
	.name			= "TCP",
	.owner			= THIS_MODULE,
	.close			= tcp_close,
	.connect		= tcp_v4_connect,
	.disconnect		= tcp_disconnect,
	.accept			= inet_csk_accept,
	.ioctl			= tcp_ioctl,
	.init			= tcp_v4_init_sock,
	.destroy		= tcp_v4_destroy_sock,
	.shutdown		= tcp_shutdown,
	.setsockopt		= tcp_setsockopt,
	.getsockopt		= tcp_getsockopt,
	.recvmsg		= tcp_recvmsg,
	.sendmsg		= tcp_sendmsg,
	.sendpage		= tcp_sendpage,
	.backlog_rcv		= tcp_v4_do_rcv,
	.release_cb		= tcp_release_cb,
	.hash			= inet_hash,
	.unhash			= inet_unhash,
	.get_port		= inet_csk_get_port,
	.enter_memory_pressure	= tcp_enter_memory_pressure,
	.stream_memory_free	= tcp_stream_memory_free,
	.sockets_allocated	= &tcp_sockets_allocated,
	.orphan_count		= &tcp_orphan_count,
	.memory_allocated	= &tcp_memory_allocated,
	.memory_pressure	= &tcp_memory_pressure,
	.sysctl_wmem		= sysctl_tcp_wmem,
	.sysctl_rmem		= sysctl_tcp_rmem,
	.max_header		= MAX_TCP_HEADER,
	.obj_size		= sizeof(struct tcp_sock),
	.slab_flags		= SLAB_DESTROY_BY_RCU,
	.twsk_prot		= &tcp_timewait_sock_ops,
	.rsk_prot		= &tcp_request_sock_ops,
	.h.hashinfo		= &tcp_hashinfo,
	.no_autobind		= true,
#ifdef CONFIG_COMPAT
	.compat_setsockopt	= compat_tcp_setsockopt,
	.compat_getsockopt	= compat_tcp_getsockopt,
#endif
#ifdef CONFIG_MEMCG_KMEM
	.init_cgroup		= tcp_init_cgroup,
	.destroy_cgroup		= tcp_destroy_cgroup,
	.proto_cgroup		= tcp_proto_cgroup,
#endif
};
```

### 3.3 `SOCK_DGRAM`操作集

`inet_dgram_ops`定义了`socket`层中`SOCK_DGRAM`类型的操作集。
```C
const struct proto_ops inet_dgram_ops = {
	.family		   = PF_INET,
	.owner		   = THIS_MODULE,
	.release	   = inet_release,
	.bind		   = inet_bind,
	.connect	   = inet_dgram_connect,
	.socketpair	   = sock_no_socketpair,
	.accept		   = sock_no_accept,
	.getname	   = inet_getname,
	.poll		   = udp_poll,
	.ioctl		   = inet_ioctl,
	.listen		   = sock_no_listen,
	.shutdown	   = inet_shutdown,
	.setsockopt	   = sock_common_setsockopt,
	.getsockopt	   = sock_common_getsockopt,
	.sendmsg	   = inet_sendmsg,
	.recvmsg	   = inet_recvmsg,
	.mmap		   = sock_no_mmap,
	.sendpage	   = inet_sendpage,
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_sock_common_setsockopt,
	.compat_getsockopt = compat_sock_common_getsockopt,
	.compat_ioctl	   = inet_compat_ioctl,
#endif
};
```

### 3.4 `UDP`操作集

`udp_prot`定义了`UDP`协议的操作集，是`SOCK_DGRAM`类型的一种具体实现。
```C
struct proto udp_prot = {
	.name		   = "UDP",
	.owner		   = THIS_MODULE,
	.close		   = udp_lib_close,
	.connect	   = ip4_datagram_connect,
	.disconnect	   = udp_disconnect,
	.ioctl		   = udp_ioctl,
	.init		   = udp_init_sock,
	.destroy	   = udp_destroy_sock,
	.setsockopt	   = udp_setsockopt,
	.getsockopt	   = udp_getsockopt,
	.sendmsg	   = udp_sendmsg,
	.recvmsg	   = udp_recvmsg,
	.sendpage	   = udp_sendpage,
	.release_cb	   = ip4_datagram_release_cb,
	.hash		   = udp_lib_hash,
	.unhash		   = udp_lib_unhash,
	.rehash		   = udp_v4_rehash,
	.get_port	   = udp_v4_get_port,
	.memory_allocated  = &udp_memory_allocated,
	.sysctl_mem	   = sysctl_udp_mem,
	.sysctl_wmem	   = &sysctl_udp_wmem_min,
	.sysctl_rmem	   = &sysctl_udp_rmem_min,
	.obj_size	   = sizeof(struct udp_sock),
	.slab_flags	   = SLAB_DESTROY_BY_RCU,
	.h.udp_table	   = &udp_table,
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_udp_setsockopt,
	.compat_getsockopt = compat_udp_getsockopt,
#endif
	.clear_sk	   = sk_prot_clear_portaddr_nulls,
};
```

## 4. 协议栈调用路径

以`TCP`的`connect`操作为例，在应用层发起一个`connect`操作，从应用层到协议实现层的调用路径如下。其它协议操作的路径也类似，了解前面的信息后，可以直接进入具体协议的函数探索实现细节和逻辑。
```Bash
Application: connect(fd, addr, len)
     ↓
System Call: sys_connect()
     ↓
Socket Layer: sock->ops->connect()
     ↓ (inet_stream_ops.connect)
inet_stream_connect()
     ↓ [Generic stream connection setup]
     ↓
Protocol Layer: sk->sk_prot->connect()
     ↓ (tcp_prot.connect)
tcp_v4_connect()
     ↓ [TCP three-way handshake]
     ↓
Network Layer
```