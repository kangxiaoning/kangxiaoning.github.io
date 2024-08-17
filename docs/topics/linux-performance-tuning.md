# Linux Performance Tuning

<show-structure depth="3"/>

记录Linux性能调优相关内容。

## 1. /proc

### 1.1 /proc/net

#### 1.1.1 /proc/net/softnet_stat

判断丢包是否是因为软中断的CPU时间不够。

打印该文件信息的函数如下，可以根据该函数了解输出字段含义。

```C
// /home/kangxiaoning/workspace/linux-3.10.0-1160/net/core/net-procfs.c

static int softnet_seq_show(struct seq_file *seq, void *v)
{
	struct softnet_data *sd = v;

	seq_printf(seq, "%08x %08x %08x %08x %08x %08x %08x %08x %08x %08x\n",
		   sd->processed, sd->dropped, sd->time_squeeze, 0,
		   0, 0, 0, 0, /* was fastroute */
		   sd->cpu_collision, sd->received_rps);
	return 0;
}
```

有含义的字段解释，部分字段直接打印0。

```Shell
Column-01: packet_process: the number of frames received by the interrupt handler.
Column-02: packet_drop: the number of frames dropped due to netdev_max_backlog being exceeded.
Column-03: time_squeeze: the number of times ksoftirqd ran out of netdev_budget or CPU time when there was still work to be done.
Column-09: cpu_collision: collision occur while obtaining device lock while transmitting.
Column-10: received_rps: number of times cpu woken up received_rps.
```

如下内容摘录自 [How to tune `net.core.netdev_max_backlog` and `net.core.netdev_budget` sysctl kernel tunables?](https://access.redhat.com/solutions/1241943)

- Each line of `softnet_stat` represents the packets a CPU core has received.
- Upon traffic coming into the NIC, a hard interrupt is generated. An IRQ handler acknowledges the hard interrupt.
- A SoftIRQ is scheduled to pull packets off the NIC at a time when it will not interrupt another process.
- The SoftIRQ runs to pull `net.core.netdev_budget` packets off the NIC
- Normally, packets are processed through network protocols as the packets are received.
- Packets may be stored in a backlog queue of approximate size `net.core.netdev_max_backlog` under certain conditions - such as when RPS is configured or the receiving device is virtual. If the `backlog` is full, the second column of `softnet_stat` will increment.
- If there were packets left when `budget` was over, the third column of `softnet_stat` will increment
- You can view `softnet_stat` in realtime whilst reproducing packet loss to determine if `backlog` and/or `budget` should be incremented
- You can change the `netdev` tunables with `sysctl -w` in realtime

如下内容摘录自 [Red Hat Enterprise Linux Network Performance Tuning Guide](https://access.redhat.com/articles/1391433)

- **time_squeeze**: If the SoftIRQs do not run for long enough, the rate of incoming data could exceed the kernel's capability to drain the buffer fast enough. As a result, the NIC buffers will overflow and traffic will be lost. Occasionally, it is necessary to increase the time that SoftIRQs are allowed to run on the CPU. This is known as the netdev_budget. The default value of the budget is 300. This will cause the SoftIRQ process to drain 300 messages from the NIC before getting off the CPU:

```Shell
# sysctl net.core.netdev_budget 
net.core.netdev_budget = 300 
```