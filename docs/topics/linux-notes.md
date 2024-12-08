# Linux碎片知识与问题拾遗

<show-structure depth="3"/>

## 1. Fedora扩容磁盘

Fedora虚拟机在创建的时候明明分配了40G，但是OS中看到的却只有20G，直到空间不够用了才发现，如下对扩容做个记录，以便后续查阅。

### 1.1 查看可用空间
```Shell
[root@localhost .cache]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/fedora_192-root   15G   11G  4.2G  73% /
devtmpfs                     4.0M     0  4.0M   0% /dev
tmpfs                        2.0G     0  2.0G   0% /dev/shm
efivarfs                     256K   32K  225K  13% /sys/firmware/efi/efivars
tmpfs                        780M  1.3M  779M   1% /run
tmpfs                        2.0G  4.0K  2.0G   1% /tmp
/dev/nvme0n1p2               960M  244M  717M  26% /boot
/dev/nvme0n1p1               599M   11M  588M   2% /boot/efi
tmpfs                        390M   12K  390M   1% /run/user/0
[root@localhost .cache]#
[root@localhost .cache]# lvs
  LV   VG         Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root fedora_192 -wi-ao---- 15.00g                                                    
[root@localhost .cache]#

# pvs查看还有23.41g可以分配
[root@localhost .cache]# pvs
  PV             VG         Fmt  Attr PSize  PFree 
  /dev/nvme0n1p3 fedora_192 lvm2 a--  38.41g 23.41g
[root@localhost .cache]#
```

### 1.2. pv扩容
```Shell
# 可用空间全部分配给root
[root@localhost .cache]# lvextend -L+23.41g /dev/mapper/fedora_192-root
  Rounding size to boundary between physical extents: 23.41 GiB.
  Size of logical volume fedora_192/root changed from 15.00 GiB (3840 extents) to 38.41 GiB (9833 extents).
  Logical volume fedora_192/root successfully resized.
[root@localhost .cache]# pvs
  PV             VG         Fmt  Attr PSize  PFree
  /dev/nvme0n1p3 fedora_192 lvm2 a--  38.41g    0 
[root@localhost .cache]# 

# 此时文件系统还是原来的大小
[root@localhost .cache]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/fedora_192-root   15G   11G  4.2G  73% /
devtmpfs                     4.0M     0  4.0M   0% /dev
tmpfs                        2.0G     0  2.0G   0% /dev/shm
efivarfs                     256K   32K  225K  13% /sys/firmware/efi/efivars
tmpfs                        780M  1.3M  779M   1% /run
tmpfs                        2.0G  4.0K  2.0G   1% /tmp
/dev/nvme0n1p2               960M  244M  717M  26% /boot
/dev/nvme0n1p1               599M   11M  588M   2% /boot/efi
tmpfs                        390M   12K  390M   1% /run/user/0
[root@localhost .cache]#
```

### 1.3. xfs容
```
[root@localhost .cache]# xfs_growfs /dev/mapper/fedora_192-root 
meta-data=/dev/mapper/fedora_192-root isize=512    agcount=4, agsize=983040 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=3932160, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 3932160 to 10068992
[root@localhost .cache]#

# 此时文件系统完成扩容
[root@localhost .cache]# df -h
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/fedora_192-root   39G   11G   28G  29% /
devtmpfs                     4.0M     0  4.0M   0% /dev
tmpfs                        2.0G     0  2.0G   0% /dev/shm
efivarfs                     256K   32K  225K  13% /sys/firmware/efi/efivars
tmpfs                        780M  1.3M  779M   1% /run
tmpfs                        2.0G  4.0K  2.0G   1% /tmp
/dev/nvme0n1p2               960M  244M  717M  26% /boot
/dev/nvme0n1p1               599M   11M  588M   2% /boot/efi
tmpfs                        390M   12K  390M   1% /run/user/0
[root@localhost .cache]#
```

## 2. NTP Server数量

NTP Server应该配置几台？

答：至少配置4台NTP Server，详情如下。

### 2.1 Using Enough Time Sources

An NTP implementation that is compliant with [RFC5905](https://www.rfc-editor.org/rfc/rfc5905.html) takes the available sources of time and submits this timing data to sophisticated intersection, clustering, and combining algorithms to get the best estimate of the correct time.  The description of these algorithms is beyond the scope of this document.  Interested readers should read [RFC5905](https://www.rfc-editor.org/rfc/rfc5905.html) or the detailed description of NTP in [MILLS2006].

- If there is only one source of time, the answer is obvious.  It may not be a good source of time, but it's the only source that can be considered.  Any issue with the time at the source will be passed on to the client.

- If there are two sources of time and they align well enough, then the best time can be calculated easily.  But if one source fails, then the solution degrades to the single-source solution outlined above.  And if the two sources don't agree, it will be difficult to know which one is correct without making use of information from outside of the protocol.

- If there are three sources of time, there is more data available to converge on the best calculated time, and this time is more likely to be accurate.  And the loss of one of the sources (by becoming unreachable or unusable) can be tolerated.  But at that point, the solution degrades to the two-source solution.

- Having four or more sources of time is better as long as the sources are diverse (Section 3.3).  If one of these sources
  develops a problem, there are still at least three other time sources.

This analysis assumes that a majority of the servers used in the solution are honest, even if some may be inaccurate.  Operators should be aware of the possibility that if an attacker is in control of the network, the time coming from all servers could be compromised.

Operators who are concerned with maintaining accurate time **SHOULD use at least four independent, diverse sources of time**.  Four sources will provide sufficient backup in case one source goes down.  If four sources are not available, operators MAY use fewer sources, which is subject to the risks outlined above.

Operators are advised to monitor all time sources that are in use. If time sources do not generally align, operators are encouraged to investigate the cause and either correct the problems or stop using defective servers.

### 2.2 参考
- [Best practices for NTP](https://access.redhat.com/solutions/778603)

## 3. 获取RedHat内核源码

下载内核源码学习及排查问题。

### 3.1 RHEL/Centos 7.9 kernel source code

在[CentOS Vault Mirror](https://vault.centos.org/7.9.2009/updates/Source/SPackages/)下载对应的版本。

```Shell
# 下载
wget https://vault.centos.org/7.9.2009/updates/Source/SPackages/kernel-3.10.0-1160.108.1.el7.src.rpm

# 提取rpm包文件
rpm2cpio ./kernel-3.10.0-1160.108.1.el7.src.rpm | cpio -idmv --directory ./kernel-3.10.0-1160
```

`kernel-3.10.0-1160/`目录下会生成`linux-3.10.0-1160.108.1.el7.tar.xz`，移动到指定位置解压即可。

> 解压后要删除`redhat/configs`文件，因为是个不存在的软件链接。
> 

```Shell
tar xf linux-3.10.0-1160.108.1.el7.tar.xz
mv linux-3.10.0-1160.108.1.el7 linux-3.10.0-1160
cd linux-3.10.0-1160/
rm -f configs
```

### 3.2 RHEL 8.8 kernel source code

在[Red Hat Packages](https://access.redhat.com/downloads/content/package-browser)搜索并下载。

比如以`kernel-4.18.0-477.27.1.el8_8.src.rpm`关键字搜索，点击查询结果会中转到如下页面，也可以在这个页面选择搜索。

[](https://access.redhat.com/downloads/content/kernel/4.18.0-477.27.1.el8_8/x86_64/fd431d51/package)

内核源码文件提取与7.9一致。

## 4. Introduction to eBPF in Red Hat Enterprise Linux 7

[Introduction to eBPF in Red Hat Enterprise Linux 7](https://www.redhat.com/en/blog/introduction-ebpf-red-hat-enterprise-linux-7)

> The eBPF in Red Hat Enterprise Linux 7.6 is provided as Tech Preview and thus doesn't come with full support and is not suitable for deployment in production.

## 5. Ubuntu 22.04 扩容磁盘

### 5.1 VMware Fusion扩容磁盘
在关机状态下扩容。

### 5.2 在Ubuntu中确认磁盘信息

```Shell
root@localhost:~# fdisk -l
Disk /dev/loop0: 77.38 MiB, 81137664 bytes, 158472 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop1: 59.75 MiB, 62652416 bytes, 122368 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop2: 59.77 MiB, 62676992 bytes, 122416 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop3: 77.38 MiB, 81133568 bytes, 158464 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop4: 33.65 MiB, 35287040 bytes, 68920 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop5: 33.71 MiB, 35344384 bytes, 69032 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
GPT PMBR size mismatch (167772159 != 251658239) will be corrected by write.


Disk /dev/nvme0n1: 120 GiB, 128849018880 bytes, 251658240 sectors # 和VMware分配的磁盘大小一致
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 93D81391-7E1D-4563-ACEB-B42E291C647B

Device           Start       End   Sectors  Size Type
/dev/nvme0n1p1    2048   2203647   2201600    1G EFI System
/dev/nvme0n1p2 2203648   6397951   4194304    2G Linux filesystem
/dev/nvme0n1p3 6397952 167770111 161372160 76.9G Linux filesystem


Disk /dev/mapper/ubuntu--vg-ubuntu--lv: 76.95 GiB, 82619400192 bytes, 161366016 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
root@localhost:~# 
```

### 5.3 扩容分区

```Shell
root@localhost:~# fdisk /dev/nvme0n1

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

GPT PMBR size mismatch (167772159 != 251658239) will be corrected by write.
This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.


Command (m for help): m # help

Help:

  GPT
   M   enter protective/hybrid MBR

  Generic
   d   delete a partition
   F   list free unpartitioned space
   l   list known partition types
   n   add a new partition
   p   print the partition table
   t   change a partition type
   v   verify the partition table
   i   print information about a partition

  Misc
   m   print this menu
   x   extra functionality (experts only)

  Script
   I   load disk layout from sfdisk script file
   O   dump disk layout to sfdisk script file

  Save & Exit
   w   write table to disk and exit
   q   quit without saving changes

  Create a new label
   g   create a new empty GPT partition table
   G   create a new empty SGI (IRIX) partition table
   o   create a new empty DOS partition table
   s   create a new empty Sun partition table


Command (m for help): n # 增加分区
Partition number (4-128, default 4):  # 回车
First sector (167770112-251658206, default 167770112):  # 回车
Last sector, +/-sectors or +/-size{K,M,G,T,P} (167770112-251658206, default 251658206):  # 回车

Created a new partition 4 of type 'Linux filesystem' and of size 40 GiB.

Command (m for help): w # 写入分区表
he partition table has been altered.
Syncing disks.

root@localhost:~# 
```

验证已经增加了新分区。

```Shell
root@localhost:~# fdisk -l
Disk /dev/loop0: 77.38 MiB, 81137664 bytes, 158472 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop1: 59.75 MiB, 62652416 bytes, 122368 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop2: 59.77 MiB, 62676992 bytes, 122416 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop3: 77.38 MiB, 81133568 bytes, 158464 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop4: 33.65 MiB, 35287040 bytes, 68920 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop5: 33.71 MiB, 35344384 bytes, 69032 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/nvme0n1: 120 GiB, 128849018880 bytes, 251658240 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 93D81391-7E1D-4563-ACEB-B42E291C647B

Device             Start       End   Sectors  Size Type
/dev/nvme0n1p1      2048   2203647   2201600    1G EFI System
/dev/nvme0n1p2   2203648   6397951   4194304    2G Linux filesystem
/dev/nvme0n1p3   6397952 167770111 161372160 76.9G Linux filesystem
/dev/nvme0n1p4 167770112 251658206  83888095   40G Linux filesystem # 新增分区


Disk /dev/mapper/ubuntu--vg-ubuntu--lv: 76.95 GiB, 82619400192 bytes, 161366016 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
root@localhost:~# 
```

### 5.4 扩容PV
```Shell
root@localhost:~# pvcreate /dev/nvme0n1p4 
  Physical volume "/dev/nvme0n1p4" successfully created.
root@localhost:~# 
root@localhost:~# vgdisplay 
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <76.95 GiB
  PE Size               4.00 MiB
  Total PE              19698
  Alloc PE / Size       19698 / <76.95 GiB
  Free  PE / Size       0 / 0   # vg可用空间为0
  VG UUID               DBHQfM-Vs4z-N3ns-dWTL-zowt-FW3J-sZb34I
   
root@localhost:~#
```

### 5.5 扩容VG

```Shell
root@localhost:~# vgextend ubuntu-vg /dev/nvme0n1p4 
  Volume group "ubuntu-vg" successfully extended
root@localhost:~# 
root@localhost:~# 
root@localhost:~# vgdisplay 
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               116.94 GiB
  PE Size               4.00 MiB
  Total PE              29937
  Alloc PE / Size       19698 / <76.95 GiB
  Free  PE / Size       10239 / <40.00 GiB # 可以看到已经扩容成功，vg可用空间为40G
  VG UUID               DBHQfM-Vs4z-N3ns-dWTL-zowt-FW3J-sZb34I
   
root@localhost:~#
```

### 5.6 扩容LV

这里使用百分比扩容，将可用空间都分配给LV。

```Shell
root@localhost:~# lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv 
  Size of logical volume ubuntu-vg/ubuntu-lv changed from <76.95 GiB (19698 extents) to 116.94 GiB (29937 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
root@localhost:~#
```

### 5.7 扩容文件系统

这里要根据具体文件系统判断要使用的命令，此处是`ext4`，使用`resize2fs`。

完成后关机，创建快照，再开机使用。

```Shell
root@localhost:~# lsblk -f
NAME                      FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop0                     squashfs    4.0                                                         0   100% /snap/lxd/29353
loop1                     squashfs    4.0                                                         0   100% /snap/core20/2321
loop2                     squashfs    4.0                                                         0   100% /snap/core20/2383
loop3                     squashfs    4.0                                                         0   100% /snap/lxd/28384
loop4                     squashfs    4.0                                                         0   100% /snap/snapd/21467
loop5                     squashfs    4.0                                                         0   100% /snap/snapd/21761
sr0                                                                                                        
nvme0n1                                                                                                    
├─nvme0n1p1               vfat        FAT32          2915-B99F                                   1G     1% /boot/efi
├─nvme0n1p2               ext4        1.0            01925585-9548-413b-a539-e3412871af3c      1.5G    13% /boot
├─nvme0n1p3               LVM2_member LVM2 001       Pafs9G-NZ5j-iLsy-ZsTv-EZL0-oOQ2-JUSMbD                
│ └─ubuntu--vg-ubuntu--lv ext4        1.0            c8df5b7d-874e-45bf-9283-ad063d01fc9e     22.5G    65% /
└─nvme0n1p4               LVM2_member LVM2 001       3HDZfU-xATg-VK13-2GwI-1EjP-gLOy-qtV3Ba                
  └─ubuntu--vg-ubuntu--lv ext4        1.0            c8df5b7d-874e-45bf-9283-ad063d01fc9e     22.5G    65% /
root@localhost:~# 
root@localhost:~# 
root@localhost:~# resize2fs /dev/ubuntu-vg/ubuntu-lv  # 扩容文件系统
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/ubuntu-vg/ubuntu-lv is mounted on /; on-line resizing required
old_desc_blocks = 10, new_desc_blocks = 15
The filesystem on /dev/ubuntu-vg/ubuntu-lv is now 30655488 (4k) blocks long.

root@localhost:~# df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              1.6G  1.3M  1.6G   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  115G   49G   61G  45% /        # 此处看到已经扩容成功
tmpfs                              7.8G     0  7.8G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/nvme0n1p2                     2.0G  251M  1.6G  14% /boot
/dev/nvme0n1p1                     1.1G  6.4M  1.1G   1% /boot/efi
tmpfs                              1.6G  4.0K  1.6G   1% /run/user/1000
root@localhost:~# sync
root@localhost:~# df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              1.6G  1.3M  1.6G   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  115G   49G   61G  45% /
tmpfs                              7.8G     0  7.8G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/nvme0n1p2                     2.0G  251M  1.6G  14% /boot
/dev/nvme0n1p1                     1.1G  6.4M  1.1G   1% /boot/efi
tmpfs                              1.6G  4.0K  1.6G   1% /run/user/1000
root@localhost:~# exit
logout
kangxiaoning@localhost:~$
```

## 6. Ubuntu debugging相关

### 6.1 获取Ubuntu源码

```bash
apt source linux-image-unsigned-$(uname -r)
```

### 6.2 Debug symbol packages

```bash
sudo apt install ubuntu-dbgsym-keyring

echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list

sudo apt-get update

sudo apt -y install linux-image-$(uname -r)-dbgsym --allow-unauthenticated
```

### 6.3 install build-essential

```Bash
sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison gcc g++ make
```