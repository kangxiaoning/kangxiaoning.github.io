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

