# CPU iowait计算原理

<show-structure depth="3"/>

> 本文基于**RHEL 3.10.0-1160**版本源码进行分析。
>
{style="note"}

## 1. Timer Interrupt

**timer interrupt**是操作系统的重要机制，用于驱动进程调度、更新系统时间、处理定时器事件、更新进程运行时间等。

`timer interrupt`由定时器周期性触发，周期间隔由`HZ`定义，默认是1秒执行100次。

```C
// /home/kangxiaoning/workspace/linux-3.10.0-1160/include/asm-generic/param.h
# define HZ		CONFIG_HZ	/* Internal kernel timer frequency */
```

```C
// /home/kangxiaoning/workspace/linux-3.10.0-1160/include/generated/autoconf.h
#define CONFIG_HZ 100
```


## 2. Periodic Tick

当CPU收到`timer interrupt`信号时，就会暂停当前任务，切换到预先定义的中断服务程序（Interrupt Service Routine），ISR就是处理`timer interrupt`这种事件的handler。

在Linux kernel中，`timer interrupt`的具体实现是**periodic tick**，`timer interrupt`这类事件对应的handler函数是`tick_handle_periodic`，具体代码如下。

```C
/*
 * Event handler for periodic ticks
 */
void tick_handle_periodic(struct clock_event_device *dev)
{
	int cpu = smp_processor_id();
	ktime_t next;

	tick_periodic(cpu);

#if defined(CONFIG_HIGH_RES_TIMERS) || defined(CONFIG_NO_HZ_COMMON)
	/*
	 * The cpu might have transitioned to HIGHRES or NOHZ mode via
	 * update_process_times() -> run_local_timers() ->
	 * hrtimer_run_queues().
	 */
	if (dev->event_handler != tick_handle_periodic)
		return;
#endif

	if (dev->mode != CLOCK_EVT_MODE_ONESHOT)
		return;
	/*
	 * Setup the next period for devices, which do not have
	 * periodic mode:
	 */
	next = ktime_add(dev->next_event, tick_period);
	for (;;) {
		if (!clockevents_program_event(dev, next, false))
			return;
		/*
		 * Have to be careful here. If we're in oneshot mode,
		 * before we call tick_periodic() in a loop, we need
		 * to be sure we're using a real hardware clocksource.
		 * Otherwise we could get trapped in an infinite
		 * loop, as the tick_periodic() increments jiffies,
		 * when then will increment time, posibly causing
		 * the loop to trigger again and again.
		 */
		if (timekeeping_valid_for_hres())
			tick_periodic(cpu);
		next = ktime_add(next, tick_period);
	}
}
```
{collapsible="true" collapsed-title="tick_handle_periodic()" default-state="collapsed"}

### 2.1 tick_periodic()

`tick_periodic()`是**Periodic tick**的核心处理函数，负责管理系统的周期性的定时事件，确保系统时间的准确性和进程调度的及时性。

其中的`update_process_times()`函数主要作用是更新进程的运行时间信息，在CPU时间片结束或发生时间更新时被调用，`iowait`通过这个函数的执行得到更新。

```C
/*
 * Periodic tick
 */
static void tick_periodic(int cpu)
{
	if (tick_do_timer_cpu == cpu) {
		write_seqlock(&jiffies_lock);

		/* Keep track of the next tick event */
		tick_next_period = ktime_add(tick_next_period, tick_period);

		do_timer(1);
		write_sequnlock(&jiffies_lock);
		update_wall_time();
	}

	update_process_times(user_mode(get_irq_regs()));
	profile_tick(CPU_PROFILING);
}
```
{collapsible="true" collapsed-title="tick_periodic()" default-state="collapsed"}

### 2.2 update_process_times()

`update_process_times()`通过传入的`user_mode(get_irq_regs())`判断处理器当前所处的模式，调用`account_process_tick(p, user_tick)`将CPU时间统计到对应的指标上，比如`top`命令中的`us`、`sy`、`id`、`wa`、`hi`、`si`等。

> user_mode(regs) 这个宏的作用是检查处理器当前是否处于用户模式。它的原理是基于处理器状态寄存器（Processor Status Register, PSR）中的特定位，这些位指示了处理器当前的操作模式。
> 
> PSR中包含多种标志位，用于反映处理器的运行状态，包括但不限于条件码、中断使能状态、处理器模式等。
> 

```C
/*
 * Called from the timer interrupt handler to charge one tick to the current
 * process.  user_tick is 1 if the tick is user time, 0 for system.
 */
void update_process_times(int user_tick)
{
	struct task_struct *p = current;
	int cpu = smp_processor_id();

	/* Note: this timer irq context must be accounted for as well. */
	account_process_tick(p, user_tick);
	run_local_timers();
	rcu_check_callbacks(cpu, user_tick);
#ifdef CONFIG_IRQ_WORK
	if (in_irq())
		irq_work_tick();
#endif
	scheduler_tick();
	run_posix_cpu_timers(p);
}
```
{collapsible="true" collapsed-title="update_process_times()" default-state="collapsed"}

### 2.3 account_process_tick()

该函数用于为进程记录CPU时间。根据参数user_tick的值，决定是用户态时间还是内核态时间。根据进程是否是空闲进程，以及是否在硬中断上下文中，分别调用不同的会计函数来记录用户态时间、内核态时间和空闲时间。

- `account_user_time()`更新该进程用户态使用的CPU时间。
- `account_system_time()`更新该进程内核态使用的CPU时间，具体包括硬中断、软中断、system消耗的CPU时间。
- `account_idle_time()`更新该进程的空闲时间，包含CPU没有被任何活动进程使用的时段，以及**等待IO的时间**。

```C
/*
 * Account a single tick of cpu time.
 * @p: the process that the cpu time gets accounted to
 * @user_tick: indicates if the tick is a user or a system tick
 */
void account_process_tick(struct task_struct *p, int user_tick)
{
	cputime_t one_jiffy_scaled = cputime_to_scaled(cputime_one_jiffy);
	struct rq *rq = this_rq();

	if (vtime_accounting_enabled())
		return;

	if (sched_clock_irqtime) {
		irqtime_account_process_tick(p, user_tick, rq);
		return;
	}

	if (steal_account_process_tick())
		return;

	if (user_tick)
		account_user_time(p, cputime_one_jiffy, one_jiffy_scaled);
	else if ((p != rq->idle) || (irq_count() != HARDIRQ_OFFSET))
		account_system_time(p, HARDIRQ_OFFSET, cputime_one_jiffy,
				    one_jiffy_scaled);
	else
		account_idle_time(cputime_one_jiffy);
}
```
{collapsible="true" collapsed-title="account_process_tick()" default-state="collapsed"}

### 2.4 account_idle_time()

`account_idle_time()`的主要功能是更新当前CPU的**空闲时间**或**等待I/O的时间**。

这里也可以看出来**等待I/O的时间**本身也属于**CPU idle**时间的一部分。
- 如果CPU当前是**空闲状态**，并且**有进程正在等待I/O**，时间将计入**iowait**。
- 如果CPU当前是**空闲状态**，并且**没有任何进程等待I/O**，时间将计入**idle**。


```C
/*
 * Account for idle time.
 * @cputime: the cpu time spent in idle wait
 */
void account_idle_time(cputime_t cputime)
{
	u64 *cpustat = kcpustat_this_cpu->cpustat;
	struct rq *rq = this_rq();

	if (atomic_read(&rq->nr_iowait) > 0)
		cpustat[CPUTIME_IOWAIT] += (__force u64) cputime;
	else
		cpustat[CPUTIME_IDLE] += (__force u64) cputime;
}
```

如果有进程正在等待I/O，但是其它进程被调度到了当前CPU上，此时该CPU在执行其它进程的指令，处于繁忙状态，代码逻辑就执行不到统计iowait的部分，所以**CPU繁忙时iowait反映不了进程等待I/O的事实**。

考虑这么一个场景：A程在等待I/O，B进程在CPU上执行，此时CPU并不是空闲状态，而是被B进程使用，这种情况下CPU时间会统计到B进程的us/sy指标上，不会统计到A进程的iowait指标，也就是当**系统中有CPU密集型任务时，即使有进程在等待I/O，iowait指标也体现不出来**。

**结论：CPU繁忙的情况下，即使有进程等待I/O，也不会体现在iowait指标上。**

## 3. CPU iowait不可靠

从上面分析可以看出来，`CPU iowait`并不是一个可靠的监控指标，`iowait`高不代表一定有I/O瓶颈，`iowait`低也不代表没有I/O瓶颈。

判断是否有进程在等待I/O，可以通过`vmstat`的`b`列来判断。

## 4. cpu.busy和cpu.iowait同时100%

**结论：监控计算逻辑错误，导出出现`cpu.busy`与`cpu.iowait`同时100%的矛盾现象。**

### 4.1 现象

在一次异常处理中，观察到`cpu.busy`与`cpu.iowait`同时100%，与预期不符。

### 4.2 矛盾

根据前面分析，`cpu.iowait`是CPU空闲时间的一部分，所以`cpu.iowait`100%表示CPU完全是空闲的，也就是`cpu.idle`是100%，那为什么`cpu.busy`也是100%呢？

### 4.3 分析

#### 4.3.1 `/proc/stat`含义

**Position 1 - user**: Time executing processes in user mode, excluding nice time. This represents normal application workload processing.

**Position 2 - nice**: Time executing niced processes (positive nice values indicating lower priority). This time is separate from regular user time.

**Position 3 - system**: Time executing in kernel/system mode, including system calls, kernel functions, and device drivers serving user processes.

**Position 4 - idle**: Time spent in idle state when the CPU has no work to perform. This is the primary metric for calculating utilization.

**Position 5 - iowait** (Linux 2.5.41+): Time waiting for I/O operations to complete. **Critical note**: This measurement is unreliable on multi-core systems because the CPU doesn't actually wait for I/O - only individual processes wait while other processes can continue running.

**Position 6 - irq** (Linux 2.6.0+): Time servicing hardware interrupts with high priority response to hardware events.

**Position 7 - softirq** (Linux 2.6.0+): Time servicing software interrupts that handle work queued by hardware interrupts at lower priority.

**Position 8 - steal** (Linux 2.6.11+): Time stolen by the hypervisor in virtualized environments, indicating CPU resources allocated to other virtual machines.

**Position 9 - guest** (Linux 2.6.24+): Time spent running virtual CPU for guest operating systems. **Critical implementation detail**: This time is also included in user time, requiring adjustment to avoid double-counting.

**Position 10 - guest_nice** (Linux 2.6.33+): Time spent running niced guest processes. **Also included in nice time**, requiring similar adjustment.

#### 4.3.2 正确计算逻辑

参考[USE Method: Linux Performance Checklist](https://www.brendangregg.com/USEmethod/use-linux.html)计算CPU Utilization。

CPU使用率是个采样指标，根据每个周期变量化计算出使用率。

1. `cpu.busy`正确计算逻辑如下：

```Shell
# Step 1: Read initial CPU statistics
prev_values = read_cpu_stats()

# Step 2: Wait for measurement interval (typically 1 second)
sleep(interval)

# Step 3: Read current CPU statistics  
curr_values = read_cpu_stats()

# Step 4: Calculate deltas
prev_idle = prev_idle + prev_iowait
curr_idle = curr_idle + curr_iowait
prev_non_idle = prev_user + prev_nice + prev_system + prev_irq + prev_softirq + prev_steal
curr_non_idle = curr_user + curr_nice + curr_system + curr_irq + curr_softirq + curr_steal

prev_total = prev_idle + prev_non_idle
curr_total = curr_idle + curr_non_idle

# Step 5: Calculate CPU usage percentage
total_delta = curr_total - prev_total
idle_delta = curr_idle - prev_idle

if total_delta == 0:
    cpu_usage = 0
else:
    cpu_usage = (total_delta - idle_delta) / total_delta * 100
```

当时ECS依赖的存储异常，I/O完全hang死，因此可以认为所有线程都在等待I/O，线程基本都处于uninterruptible sleep状态（即D状态），`user` `nice` `system` `idle` `irq` `softirq` `steal` `guest` 列的变化量基本为0，但是`iowait`变化较大，代入上述逻辑计算如下。

```Shell
# 假设上次idle列统计值为x
prev_idle = x

# 假设上次iowait列统计值为y
prev_iowait = y 

# 因为I/O不可用，所有进程处于uninterruptible sleep状态（即D状态），所有空闲时间会计入iowait中，idle列变化基本为0
curr_idle = x + 0

# 因为I/O不可用，所有进程处于uninterruptible sleep状态（即D状态），所有空闲时间会计入iowait中，假设iowait列变化为z
prev_iowait = y + z


prev_idle = prev_idle + prev_iowait = x + y
curr_idle = curr_idle + curr_iowait = x + y + z

# 假设上次采集的user nice system irq softirq steal数值分别为a b c d e f
prev_non_idle = prev_user + prev_nice + prev_system + prev_irq + prev_softirq + prev_steal
              = a + b + c + d + e + f
# user nice system irq softirq steal变化量基本为0
curr_non_idle = curr_user + curr_nice + curr_system + curr_irq + curr_softirq + curr_steal
              = (a+0) + (b+0) + (c+0) + (d+0) + (e+0) + (f+0)
              = a + b + c + d + e + f
              
prev_total = prev_idle + prev_non_idle = (x + y) + (a + b + c + d + e + f)
curr_total = curr_idle + curr_non_idle = (x + y + z) + (a + b + c + d + e + f)          

total_delta = curr_total - prev_total
            = ((x + y + z) + (a + b + c + d + e + f)) - ((x + y) + (a + b + c + d + e + f)) 
            = z
            
idle_delta = curr_idle - prev_idle
           = (x + y + z) - (x + y)
           = z

# cpu.busy
cpu_usage = (total_delta - idle_delta) / total_delta * 100
          = (z - z)/z * 100
          = 0
```

根据正确的逻辑，`cpu.busy` 的计算结果应该是0%，符合预期。

2. `cpu.iowait`正确计算逻辑如下：

```Shell
iowait_delta = z

total_delta = curr_total - prev_total
            = ((x + y + z) + (a + b + c + d + e + f)) - ((x + y) + (a + b + c + d + e + f)) 
            = z

# cpu.iowait
iowait_delta = iowait_delta / total_delta * 100
             = z / z * 100
             = 100
```

根据正确的逻辑，`cpu.iowait` 的计算结果应该是100%，符合预期。

#### 4.3.3 错误计算逻辑

查看监控指标说明，看到如下描述：
- `cpu.busy`: 100.0减去`cpu.idle`
- `cpu.idle`: `/proc/stat`中第5列的数值在采集周期内某一秒的增量与总量(2至10列之和)的百分比。
- `cpu.iowait`: `/proc/stat`中第6列的数值在采集周期内某一秒的增量与总量(2至10列之和)的百分比。

1. `cpu.idle`: 根据4.3.2的分析，第5列的增量基本为0，因此无论分母是多少，结果都是0，也就是`cpu.idle`的值基本是0%，与监控观察的结果一致。
2. `cpu.busy`: 100.0减去`cpu.idle`，可以得出`cpu.busy`的结果是100%，与观察到的结果一致。
3. `cpu.iowait`计算逻辑如下，得出结果为100%，与观察到的结果一致。。

根据4.3.2的分析， 第6列的增量为z，即分子为z，计算逻辑如下：

```Shell
# 分母
total_delta = 2至10列增量之和
            = 0 + 0 + 0 + 0 + z + 0 + 0 + 0 + 0
            = z

#  第6列的增量为z
cpu.iowait = `/proc/stat`中第6列的数值在采集周期内某一秒的增量与总量(2至10列之和)的百分比
           = z / z
           = 100%
```

根据上述分析，错误逻辑主要会影响`cpu.idle`和`cpu.busy`，而`cpu.iowait`不受影响。在`cpu.iowait`较高的场景下，`cpu.busy`计算结果比实际高，可能会提前触发告警(实际影响基本可以忽略)，而CPU真正的idle需要将`cpu.idle`和`cpu.iowait`加起来。

## 参考

[Linux's iowait statistic and multi-CPU machines](https://utcc.utoronto.ca/~cks/space/blog/linux/LinuxMultiCPUIowait)

[Understanding Linux IOWait](https://www.percona.com/blog/understanding-linux-iowait/)