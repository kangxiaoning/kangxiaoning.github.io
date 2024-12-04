# Debugging the Linux kernel with GDB

<show-structure depth="3"/>

`gdb`调试Linux kernel的一些记录。

## 1. debugging the kernel module

```Bash
# 加载模块
insmod spfs.ko

# 获取地址
cd /sys/module/spfs/sections && sudo cat .text .rodata .data .bss

# 添加symbol
add-symbol-file /lib/modules/6.8.0-49-generic/updates/spfs.ko 0xffff80007c093000 -s .rodata 0xffff80007c09b028 -s .data 0xffff80007c097180 -s .bss 0xffff80007c098240
```

```Bash
(gdb) info b
No breakpoints, watchpoints, tracepoints, or catchpoints.
(gdb) br spfs_mount
Breakpoint 1 at 0xffff80007c094d58: file /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c, line 424.
(gdb) br spfs_fill_super
Breakpoint 2 at 0xffff80007c095550: file /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c, line 332.
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0xffff80007c094d58 in spfs_mount at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:424
2       breakpoint     keep y   0xffff80007c095550 in spfs_fill_super at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:332
(gdb) c
Continuing.

Breakpoint 1, spfs_mount (fs_type=0xffff80007c0979e0 <spfs_fs_type>, flags=0, dev_name=0xffff00008aa47320 "/dev/loop0", data=0x0 <exit_spfs_fs>)
    at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:424
424	{
(gdb) bt
#0  spfs_mount (fs_type=0xffff80007c0979e0 <spfs_fs_type>, flags=0, dev_name=0xffff00008aa47320 "/dev/loop0", data=0x0 <exit_spfs_fs>) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:424
#1  0xffff80008057ea3c in legacy_get_tree (fc=0xffff00008a2943c0) at /build/linux-goHVUM/linux-6.8.0/fs/fs_context.c:662
#2  0x48c480008051e344 in ?? ()
#3  0x00000000ffffffff in ?? ()
Backtrace stopped: previous frame identical to this frame (corrupt stack?)
(gdb) delete 1
(gdb) c
Continuing.

Breakpoint 2, spfs_fill_super (sb=0xffff00008a3a5000, data=0x0 <exit_spfs_fs>, silent=0) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:332
332	{
(gdb) bt
#0  spfs_fill_super (sb=0xffff00008a3a5000, data=0x0 <exit_spfs_fs>, silent=0) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:332
#1  0xffff80008052166c in mount_bdev (fs_type=<optimized out>, flags=268435456, dev_name=<optimized out>, data=0x0 <exit_spfs_fs>, fill_super=0xffff80007c095550 <spfs_fill_super>)
    at /build/linux-goHVUM/linux-6.8.0/fs/super.c:1667
#2  0x24a780007c094db0 in ?? ()
#3  0xffff00008a2943c0 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
(gdb) p super_blocks
$4 = {
  next = 0xffff000080277800,
  prev = 0xffff00008a3a5000
}
(gdb) delete 2
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
cpu_do_idle () at /build/linux-goHVUM/linux-6.8.0/arch/arm64/kernel/idle.c:32
32		arm_cpuidle_restore_irq_context(&context);
(gdb) p super_blocks
$9 = {
  next = 0xffff000080277800,
  prev = 0xffff00008a3a5000
}
(gdb) pipe p *(struct super_block *)0xffff00008a3a5000 | grep fs_info
  s_fs_info = 0xffff00008522e000,
(gdb) p *(struct spfs_sb_info *)0xffff00008522e000
$11 = {
  s_nifree = 123,
  s_inode = {1, 1, 1, 1, 1, 0 <repeats 123 times>},
  s_nbfree = 626,
  s_block = {1, 1, 1, 1, 1, 0 <repeats 755 times>},
  s_lock = {
    owner = {
      counter = 0
    },
    wait_lock = {
      raw_lock = {
        {
          val = {
            counter = 0
          },
          {
            locked = 0 '\000',
            pending = 0 '\000'
          },
          {
            locked_pending = 0,
            tail = 0
          }
        }
      }
    },
    osq = {
      tail = {
        counter = 0
      }
    },
    wait_list = {
      next = 0xffff00008522fbe0,
      prev = 0xffff00008522fbe0
    }
  }
}
(gdb) 
```
{collapsible="true" collapsed-title="debuging filesystem" default-state="collapsed"}

## 2. 同名symbol处理

`disable`掉不需要的`breakpoint`。

```Bash
(gdb) br sp_lookup
Breakpoint 8 at 0xffff80007c093508: sp_lookup. (2 locations)
(gdb) info b
Num     Type           Disp Enb Address            What
8       breakpoint     keep y   <MULTIPLE>         
8.1                         y   0xffff80007c093508 in sp_lookup at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:394
8.2                         y   0xffff8000804b41f0 in sp_lookup at /build/linux-goHVUM/linux-6.8.0/mm/mempolicy.c:2386
(gdb) disable 8.2
(gdb) info b
Num     Type           Disp Enb Address            What
8       breakpoint     keep y   <MULTIPLE>         
8.1                         y   0xffff80007c093508 in sp_lookup at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:394
8.2                         n   0xffff8000804b41f0 in sp_lookup at /build/linux-goHVUM/linux-6.8.0/mm/mempolicy.c:2386
(gdb)
```

## 3. next指令异常

**问题现象**：执行`next`指令，没有按预期执行到下一行，而执行到其它汇编代码。

```Bash
(gdb) b 367
Breakpoint 2 at 0xffff80007c095650: file /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c, line 367.
(gdb) c
Continuing.

Breakpoint 1, spfs_fill_super (sb=0xffff00008a3a5000, data=0x0 <exit_spfs_fs>, silent=0) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:332
332	{
(gdb) p *spfs_sb
value has been optimized out
(gdb) n
vectors () at /build/linux-goHVUM/linux-6.8.0/arch/arm64/kernel/entry.S:526
526		kernel_ventry	1, h, 64, irq		// IRQ EL1h
(gdb) p *spfs_sb
No symbol "spfs_sb" in current context.
(gdb) n
el1h_64_irq () at /build/linux-goHVUM/linux-6.8.0/arch/arm64/kernel/entry.S:594
594		entry_handler	1, h, 64, irq
(gdb) n
ret_to_kernel () at /build/linux-goHVUM/linux-6.8.0/arch/arm64/kernel/entry.S:609
609		kernel_exit 1
(gdb) n
handle_softirqs (ksirqd=<optimized out>) at /build/linux-goHVUM/linux-6.8.0/kernel/softirq.c:553
warning: Source file is more recent than executable.
553			h->action(h);
(gdb) n
vectors () at /build/linux-goHVUM/linux-6.8.0/arch/arm64/kernel/entry.S:526
526		kernel_ventry	1, h, 64, irq		// IRQ EL1h
(gdb) n
el1h_64_irq () at /build/linux-goHVUM/linux-6.8.0/arch/arm64/kernel/entry.S:594
594		entry_handler	1, h, 64, irq
(gdb) n

```

**解决方法**：在要停止的地上打`breakpoint`，删除之前的`breakpoint`，然后`continue`。

```Bash
(gdb) info b
Num     Type           Disp Enb Address            What
9       breakpoint     keep y   0xffff80007c093540 in sp_lookup at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:399
	breakpoint already hit 2 times
10      breakpoint     keep y   0xffff80007c0935a8 in sp_lookup at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:407
(gdb) delete 9
(gdb) c
Continuing.

Breakpoint 10, sp_lookup (dip=<optimized out>, dentry=0xffff000083fc09c0, flags=<optimized out>) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:407
407			printk("spfs: sp_lookup name = %s, inode = %px (ino=%d)\n",
(gdb) p inum
$12 = 4
(gdb)
# 注意这里执行continue还是会执行到之前的breakpoint
(gdb) c
Continuing.

Breakpoint 10, sp_lookup (dip=<optimized out>, dentry=0xffff000083fc09c0, flags=<optimized out>) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:407
407			printk("spfs: sp_lookup name = %s, inode = %px (ino=%d)\n",
(gdb) info b
Num     Type           Disp Enb Address            What
10      breakpoint     keep y   0xffff80007c0935a8 in sp_lookup at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_dir.c:407
	breakpoint already hit 2 times
(gdb)
# 删除breakpoint后再执行continue才正常
(gdb) delete 10
(gdb) c
Continuing.

```