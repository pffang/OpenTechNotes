[TOC]

M11SDV-8C+-LN4这块板子SATA控制器不带RAID功能，不过没关系，可以利用LVM强大的功能来实现。这里使用了LVM本身的RAID和Cache功能。
一般Ubuntu应该以及安装了lvm软件包，如果没有，执行`sudo apt install lvm2`安装一下。因为启用了LVM Cache所以要执行`sudo apt install thin-provisioning-tools`安装依赖，不然Cache logic volume会失败。

# 枚举当前系统中的块设备
```bash
user@Gateway:~$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0         7:0    0  88.7M  1 loop /snap/core/7396
sda           8:0    1  55.9G  0 disk
├─sda1        8:1    1   512M  0 part /boot/efi
└─sda2        8:2    1  55.4G  0 part /
sdb           8:16   1 119.2G  0 disk
├─sdb1        8:17   1   499M  0 part
├─sdb2        8:18   1   100M  0 part
├─sdb3        8:19   1    16M  0 part
└─sdb4        8:20   1 118.7G  0 part
sdc           8:32   1   2.7T  0 disk
└─sdc1        8:33   1   2.7T  0 part
sdd           8:48   1   2.7T  0 disk
└─sdd1        8:49   1   2.7T  0 part
nvme0n1     259:0    0 238.5G  0 disk
└─nvme0n1p1 259:1    0 238.5G  0 part
```
我一共安装了5块硬盘：
- sda 一个60G SSD作为当前Ubuntu Server的安装盘
- sdb 一个128G SSD作为Windows Server的安装盘
- sdc和sdd 两个3T的HDD作为数据存储
- nvme0n1 一块256G海康威视的nvme SSD作为Cache
sdc、sdd和nvme0n1事先都使用parted建立了GPT分区表，每个盘全部都分为一个分区。
虽然LVM无需分区可以直接把整个硬盘作为物理卷，但是这样的话，在WindowsServer里面这几个盘会因为没有分区表被Windows提示没有分区表是否初始化。万一手滑点了初始化就全完了。所以这里建好分区表和分区时为了兼容性。

# 建立物理卷
```bash
user@Gateway:~$ sudo pvcreate /dev/sdc1 /dev/sdd1 /dev/nvme0n1p1
 Physical volume "/dev/sdc1" successfully created.
 Physical volume "/dev/sdd1" successfully created.
 Physical volume "/dev/nvme0n1p1" successfully created.

user@Gateway:~$ sudo pvdisplay #查看建立好的物理卷，不是必需步骤
```

# 建立卷组
```bash
user@Gateway:~$ sudo vgcreate VG_DATA /dev/sdc1 /dev/sdd1 /dev/nvme0n1p1
 Volume group "VG_DATA" successfully created
```
# 建立RAID逻辑卷
这里是建立一个逻辑卷，两个物理卷用RAID1 模式互为镜像。格式化为XFS文件系统 。当然，EXT4、ZFS什么的也是可以的。
```bash
user@Gateway:~$ sudo lvcreate --mirrors 1 --type raid1 -l 100%FREE --nosync -n LV_DATA VG_DATA
 WARNING: New raid1 won't be synchronised. Don't read what you didn't write!
 Logical volume "LV_DATA" created.

user@Gateway:~$ sudo mkfs.xfs -f /dev/VG_DATA/LV_DATA
```

# 建立Cache
lvm的Cache建立还是有点复杂的，一个cache pool包含data和metedata两部分。具体内容建议阅读lvm官方文档，或者谷歌之。
不过也有偷懒方式，可以直接创建cache-pool，把cache data和cache metadata一次性创建在单个物理卷上。
默认cachemode是writethrough，这里改成writeback 提高性能，但是可能会丢失数据。
### 创建Cache Pool
```bash
user@Gateway:~$ sudo lvcreate --type cache-pool --cachemode writeback --name LV_CachePool -l 100%FREE VG_DATA /dev/nvme0n1p1
 Using 256.00 KiB chunk size instead of default 64.00 KiB, so cache pool has less than 1000000 chunks.
 Logical volume "LV_CachePool" created.
```
### 将Cache Pool和需要cache的逻辑卷建立连接
```bash
user@Gateway:~$ sudo lvconvert --type cache --cachepool VG_DATA/LV_CachePool VG_DATA/LV_DATA
Do you want wipe existing metadata of cache pool VG_DATA/LV_CachePool? [y/n]: y
 WARNING: Data redundancy could be lost with writeback caching of raid logical volume!
 Logical volume VG_DATA/LV_DATA is now cached.
```

大功告成，看一下当前逻辑卷情况
```bash
user@Gateway:~$ sudo lvdisplay
  --- Logical volume ---
  LV Path                /dev/VG_DATA/LV_DATA
  LV Name                LV_DATA
  VG Name                VG_DATA
  LV UUID                Ck5Iqh-sstr-1PCk-5j7K-ZYFS-XWa9-qFMXNu
  LV Write Access        read/write
  LV Creation host, time Gateway, 2019-07-31 19:49:45 +0800
  LV Cache pool name     LV_CachePool
  LV Cache origin name   LV_DATA_corig
  LV Status              available
  # open                 0
  LV Size                <2.73 TiB
  Cache used blocks      99.99%
  Cache metadata blocks  23.93%
  Cache dirty blocks     99.50%
  Cache read hits/misses 16376 / 3743
  Cache wrt hits/misses  67795 / 2649117
  Cache demotions        0
  Cache promotions       0
  Current LE             715395
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:7
```

# 将LVM卷加入fstab
查看这个带cache的逻辑卷的UUID
```bash
user@Gateway:~$ sudo blkid /dev/VG_DATA/LV_DATA
/dev/VG_DATA/LV_DATA: UUID="e3d7f55c-989a-49c5-b220-704e615d0878" TYPE="xfs"
```
建立挂载点文件夹`sudo mkdir /mnt/lvmVolume`
在/etc/fstab加入
`UUID=e3d7f55c-989a-49c5-b220-704e615d0878 /mnt/lvmVolume xfs defaults 0 2`
`sudo mount -a` 立刻挂载
`sudo chown user:user /mnt/lvmVolume`将卷所有者改为user

# 重装系统之后激活以前的LVM卷
重新安装的系统要重新激活磁盘上已经存在的lvm执行
```bash
sudo vgchange -ay
```