# Fedora扩容磁盘

Fedora虚拟机在创建的时候明明分配了40G，但是OS中看到的却只有20G，直到空间不够用了才发现，如下对扩容做个记录，以便后续查阅。

## 1. 查看可用空间
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

## 2. pv扩容
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

## 3. xfs容
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