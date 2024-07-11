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
- 如果CPU当前是空闲状态，并且没有任何工作要做，时间将计入**idle**。
- 如果CPU当前是空闲状态，并且正在等待**I/O**，时间将计入**iowait**。

但是如果进程正在等待I/O，但是其它进程被调度到了CPU上，此时CPU不是空闲状态，因此等待I/O的时间不会计入**iowait**，也就是**CPU繁忙的情况下，即使有进程等待I/O，也不会体现在iowait** 指标上。

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
## 3. CPU iowait不可靠

从上面分析可以看出来，`CPU iowait`并不是一个可靠的监控指标，`iowait`高不代表一定有I/O瓶颈，`iowait`低也不代表没有I/O瓶颈。

判断是否有进程在等待I/O，可以通过`vmstat`的`b`列来判断。

## 参考

[Linux's iowait statistic and multi-CPU machines](https://utcc.utoronto.ca/~cks/space/blog/linux/LinuxMultiCPUIowait)

[Understanding Linux IOWait](https://www.percona.com/blog/understanding-linux-iowait/)