# Debugging the Linux kernel with GDB

<show-structure depth="3"/>

`gdb`调试Linux kernel的一些记录。

## 1. .gdbinit配置

通过如下设置，每次只需要执行`gdb`就可以使用了，省去一些重复操作。

```Bash
kangxiaoning@localhost:~$ more .gdb/spfs-symbols.sh 
#!/bin/bash

MOD=/lib/modules/6.8.0-49-generic/updates/spfs.ko
SECTIONS_DIR=/sys/module/spfs/sections
USR=kangxiaoning@192.168.166.35

ssh $USR sudo insmod $MOD 2>/dev/null

text=$(ssh $USR sudo cat $SECTIONS_DIR/.text)
rodata=$(ssh $USR sudo cat $SECTIONS_DIR/.rodata)
data=$(ssh $USR sudo cat $SECTIONS_DIR/.data)
bss=$(ssh $USR sudo cat $SECTIONS_DIR/.bss)

echo "add-symbol-file $MOD $text -s .rodata $rodata -s .data $data -s .bss $bss"

kangxiaoning@localhost:~$
```
{collapsible="true" collapsed-title=".gdb/spfs-symbols.sh" default-state="collapsed"}

```Bash
sudo EDITOR=vim visudo

# 最后一行添加如下内容
kangxiaoning ALL=(ALL) NOPASSWD: ALL
```
{collapsible="true" collapsed-title="sudo EDITOR=vim visudo" default-state="collapsed"}

```Bash
kangxiaoning@localhost:~$ more .gdbinit 
# connect to target
define connect_to_target
    target remote tcp:192.168.166.1:8865
end

# add spfs symbols
define add_spfs_symbol
    set confirm off
    shell ~/.gdb/spfs-symbols.sh > /tmp/spfs-symbols.gdb
    source /tmp/spfs-symbols.gdb
    set confirm on
end

# add symbol file
add-symbol-file /usr/lib/debug/boot/vmlinux-6.8.0-49-generic

# pretty print
set print pretty on

# linux kernel helper function
add-auto-load-safe-path /home/kangxiaoning/workspace/linux-6.8.0/scripts/gdb/vmlinux-gdb.py

# replace directory
set substitute-path /build/linux-goHVUM /home/kangxiaoning/workspace

add_spfs_symbol
connect_to_target
kangxiaoning@localhost:~$ 
```
{collapsible="true" collapsed-title=".gdbinit" default-state="collapsed"}

如下是debugging过程示例，在ARM平台的`call stack`不完整，x86-64的`call stack`是完整的。

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
{collapsible="true" collapsed-title="debuging filesystem on arm64" default-state="collapsed"}

```Bash
kangxiaoning@localhost:~$ gdb
GNU gdb (Ubuntu 15.0.50.20240403-0ubuntu1) 15.0.50.20240403-git
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
add symbol table from file "/usr/lib/debug/boot/vmlinux-6.8.0-49-generic"
add symbol table from file "/lib/modules/6.8.0-49-generic/updates/spfs.ko" at
	.text_addr = 0xc0982000
	.rodata_addr = 0xc098a058
	.data_addr = 0xc0986180
	.bss_addr = 0xc0987300

warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
vsnprintf (buf=0xffffc90000243d4d "", buf@entry=0xffffc90000243cfd "0\031\005@=$", size=<optimized out>, size@entry=11, fmt=fmt@entry=0xffffffff82c110e7 "%u", args=args@entry=0xffffc90000243ce0)
    at /build/linux-uoESLx/linux-6.8.0/lib/vsprintf.c:2778

This GDB supports auto-downloading debuginfo from the following URLs:
  <https://debuginfod.ubuntu.com>
Enable debuginfod for this session? (y or [n]) [answered N; input not from terminal]
Debuginfod has been disabled.
To make this setting permanent, add 'set debuginfod enabled off' to .gdbinit.

warning: 2778	/build/linux-uoESLx/linux-6.8.0/lib/vsprintf.c: No such file or directory
(gdb) br spfs_mount
Breakpoint 1 at 0xffffffffc0983940: file /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c, line 424.
(gdb) c
Continuing.

Breakpoint 1, spfs_mount (fs_type=0xffffffffc0986ca0 <spfs_fs_type>, flags=0, dev_name=0xffff88810573ab30 "/dev/loop0", data=0x0 <fixed_percpu_data>)
    at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:424
(gdb) bt 5
#0  spfs_mount (fs_type=0xffffffffc0986ca0 <spfs_fs_type>, flags=0, dev_name=0xffff88810573ab30 "/dev/loop0", data=0x0 <fixed_percpu_data>) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:424
#1  0xffffffff8153bebb in legacy_get_tree (fc=0xffff8881090969c0) at /build/linux-uoESLx/linux-6.8.0/fs/fs_context.c:662
#2  0xffffffff814e790a in vfs_get_tree (fc=fc@entry=0xffff8881090969c0) at /build/linux-uoESLx/linux-6.8.0/fs/super.c:1788
#3  0xffffffff8151c6d0 in do_new_mount (path=path@entry=0xffffc90000a13b60, fstype=fstype@entry=0xffff8881001c0900 "spfs", sb_flags=sb_flags@entry=0, mnt_flags=<optimized out>, 
    name=name@entry=0xffff8881000544c0 "/dev/loop0", data=data@entry=0x0 <fixed_percpu_data>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3352
#4  0xffffffff8151d5c0 in path_mount (dev_name=dev_name@entry=0xffff8881000544c0 "/dev/loop0", path=path@entry=0xffffc90000a13b60, type_page=type_page@entry=0xffff8881001c0900 "spfs", flags=<optimized out>, 
    flags@entry=0, data_page=data_page@entry=0x0 <fixed_percpu_data>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3679
(More stack frames follow...)
(gdb) bt
#0  spfs_mount (fs_type=0xffffffffc0986ca0 <spfs_fs_type>, flags=0, dev_name=0xffff88810573ab30 "/dev/loop0", data=0x0 <fixed_percpu_data>) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:424
#1  0xffffffff8153bebb in legacy_get_tree (fc=0xffff8881090969c0) at /build/linux-uoESLx/linux-6.8.0/fs/fs_context.c:662
#2  0xffffffff814e790a in vfs_get_tree (fc=fc@entry=0xffff8881090969c0) at /build/linux-uoESLx/linux-6.8.0/fs/super.c:1788
#3  0xffffffff8151c6d0 in do_new_mount (path=path@entry=0xffffc90000a13b60, fstype=fstype@entry=0xffff8881001c0900 "spfs", sb_flags=sb_flags@entry=0, mnt_flags=<optimized out>, 
    name=name@entry=0xffff8881000544c0 "/dev/loop0", data=data@entry=0x0 <fixed_percpu_data>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3352
#4  0xffffffff8151d5c0 in path_mount (dev_name=dev_name@entry=0xffff8881000544c0 "/dev/loop0", path=path@entry=0xffffc90000a13b60, type_page=type_page@entry=0xffff8881001c0900 "spfs", flags=<optimized out>, 
    flags@entry=0, data_page=data_page@entry=0x0 <fixed_percpu_data>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3679
#5  0xffffffff8151dd47 in do_mount (data_page=0x0 <fixed_percpu_data>, flags=0, type_page=0xffff8881001c0900 "spfs", dir_name=<optimized out>, dev_name=0xffff8881000544c0 "/dev/loop0")
    at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3692
#6  __do_sys_mount (data=<optimized out>, flags=0, type=<optimized out>, dir_name=<optimized out>, dev_name=<optimized out>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3898
#7  __se_sys_mount (data=<optimized out>, flags=0, type=<optimized out>, dir_name=<optimized out>, dev_name=<optimized out>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3875
#8  __x64_sys_mount (regs=<optimized out>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3875
#9  0xffffffff810072b5 in x64_sys_call (regs=regs@entry=0xffffc90000a13f58, nr=nr@entry=165) at ./arch/x86/include/generated/asm/syscalls_64.h:166
#10 0xffffffff8222601f in do_syscall_x64 (nr=165, regs=0xffffc90000a13f58) at /build/linux-uoESLx/linux-6.8.0/arch/x86/entry/common.c:52
#11 do_syscall_64 (regs=0xffffc90000a13f58, nr=165) at /build/linux-uoESLx/linux-6.8.0/arch/x86/entry/common.c:83
#12 0xffffffff82400130 in entry_SYSCALL_64 () at /build/linux-uoESLx/linux-6.8.0/arch/x86/entry/entry_64.S:121
#13 0x000065252c49cc60 in ?? ()
#14 0x000065252c49cf10 in ?? ()
#15 0x000065252c49cf50 in ?? ()
#16 0x000065252c49cf30 in ?? ()
#17 0x00007ffdf950b190 in ?? ()
#18 0x000065252c49cb00 in ?? ()
#19 0x0000000000000246 in ?? ()
#20 0x0000000000000000 in ?? ()
(gdb) br spfs_fill_super
Breakpoint 2 at 0xffffffffc09840b0: file /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c, line 332.
(gdb) c
Continuing.

Breakpoint 2, spfs_fill_super (sb=0xffff888102851800, data=0x0 <fixed_percpu_data>, silent=0) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:332
(gdb) bt
#0  spfs_fill_super (sb=0xffff888102851800, data=0x0 <fixed_percpu_data>, silent=0) at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:332
#1  0xffffffff814ea126 in mount_bdev (fs_type=fs_type@entry=0xffffffffc0986ca0 <spfs_fs_type>, flags=268435456, flags@entry=0, dev_name=dev_name@entry=0xffff88810573ab30 "/dev/loop0", 
    data=data@entry=0x0 <fixed_percpu_data>, fill_super=fill_super@entry=0xffffffffc09840b0 <spfs_fill_super>) at /build/linux-uoESLx/linux-6.8.0/fs/super.c:1667
#2  0xffffffffc0983980 in spfs_mount (fs_type=0xffffffffc0986ca0 <spfs_fs_type>, flags=0, dev_name=0xffff88810573ab30 "/dev/loop0", data=0x0 <fixed_percpu_data>)
    at /home/kangxiaoning/workspace/spfs/ubuntu/24.04/kern/sp_inode.c:426
#3  0xffffffff8153bebb in legacy_get_tree (fc=0xffff8881090969c0) at /build/linux-uoESLx/linux-6.8.0/fs/fs_context.c:662
#4  0xffffffff814e790a in vfs_get_tree (fc=fc@entry=0xffff8881090969c0) at /build/linux-uoESLx/linux-6.8.0/fs/super.c:1788
#5  0xffffffff8151c6d0 in do_new_mount (path=path@entry=0xffffc90000a13b60, fstype=fstype@entry=0xffff8881001c0900 "spfs", sb_flags=sb_flags@entry=0, mnt_flags=<optimized out>, 
    name=name@entry=0xffff8881000544c0 "/dev/loop0", data=data@entry=0x0 <fixed_percpu_data>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3352
#6  0xffffffff8151d5c0 in path_mount (dev_name=dev_name@entry=0xffff8881000544c0 "/dev/loop0", path=path@entry=0xffffc90000a13b60, type_page=type_page@entry=0xffff8881001c0900 "spfs", flags=<optimized out>, 
    flags@entry=0, data_page=data_page@entry=0x0 <fixed_percpu_data>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3679
#7  0xffffffff8151dd47 in do_mount (data_page=0x0 <fixed_percpu_data>, flags=0, type_page=0xffff8881001c0900 "spfs", dir_name=<optimized out>, dev_name=0xffff8881000544c0 "/dev/loop0")
    at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3692
#8  __do_sys_mount (data=<optimized out>, flags=0, type=<optimized out>, dir_name=<optimized out>, dev_name=<optimized out>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3898
#9  __se_sys_mount (data=<optimized out>, flags=0, type=<optimized out>, dir_name=<optimized out>, dev_name=<optimized out>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3875
#10 __x64_sys_mount (regs=<optimized out>) at /build/linux-uoESLx/linux-6.8.0/fs/namespace.c:3875
#11 0xffffffff810072b5 in x64_sys_call (regs=regs@entry=0xffffc90000a13f58, nr=nr@entry=165) at ./arch/x86/include/generated/asm/syscalls_64.h:166
#12 0xffffffff8222601f in do_syscall_x64 (nr=165, regs=0xffffc90000a13f58) at /build/linux-uoESLx/linux-6.8.0/arch/x86/entry/common.c:52
#13 do_syscall_64 (regs=0xffffc90000a13f58, nr=165) at /build/linux-uoESLx/linux-6.8.0/arch/x86/entry/common.c:83
#14 0xffffffff82400130 in entry_SYSCALL_64 () at /build/linux-uoESLx/linux-6.8.0/arch/x86/entry/entry_64.S:121
#15 0x000065252c49cc60 in ?? ()
#16 0x000065252c49cf10 in ?? ()
#17 0x000065252c49cf50 in ?? ()
#18 0x000065252c49cf30 in ?? ()
#19 0x00007ffdf950b190 in ?? ()
#20 0x000065252c49cb00 in ?? ()
#21 0x0000000000000246 in ?? ()
#22 0x0000000000000000 in ?? ()
(gdb) 
(gdb) p file_systems
$1 = (struct file_system_type *) 0xffffffff83675b20 <sysfs_fs_type>
(gdb) p file_systems.name
$2 = 0xffffffff82c0d0b9 "sysfs"
(gdb) p file_systems->next.name
$3 = 0xffffffff82c1cd62 "tmpfs"
(gdb) p file_systems->next->next.name
$4 = 0xffffffff82c2a446 "bdev"
(gdb) p super_blocks
$5 = {
  next = 0xffff8881002f6000,
  prev = 0xffff888102851800
}
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
0xffffffff8222ce5b in pv_native_safe_halt () at /build/linux-uoESLx/linux-6.8.0/arch/x86/kernel/paravirt.c:128
warning: 128	/build/linux-uoESLx/linux-6.8.0/arch/x86/kernel/paravirt.c: No such file or directory
(gdb) p super_blocks
$6 = {
  next = 0xffff8881002f6000,
  prev = 0xffff888102851800
}
(gdb) pipe p *(struct super_block *)0xffff888102851800 | grep fs_info
  s_fs_info = 0xffff888103888000,
(gdb) p *(struct spfs_sb_info *)0xffff888103888000
$8 = {
  s_nifree = 124,
  s_inode = {1, 1, 1, 1, 0 <repeats 124 times>},
  s_nbfree = 629,
  s_block = {1, 1, 0 <repeats 758 times>},
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
      next = 0xffff888103889be0,
      prev = 0xffff888103889be0
    }
  }
}
(gdb)
```
{collapsible="true" collapsed-title="debuging filesystem on x86-64" default-state="collapsed"}


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