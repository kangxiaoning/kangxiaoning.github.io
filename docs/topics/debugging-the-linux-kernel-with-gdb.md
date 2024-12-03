# Debugging the Linux kernel with GDB

<show-structure depth="3"/>

`gdb`调试Linux kernel的一些记录。

## 1. debugging the kernel module

```Bash
# 加载模块
insmod spfs.ko

# 获取地址
sudo grep -e spfs /proc/modules

# 添加symbol
add-symbol-file /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/spfs.ko 0xffff80007c093000
```

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