## 系统初始化

### 命令行启动

```shell
# 将图形启动改为命令行启动
sudo mv /etc/systemd/system/default.target /etc/systemd/system/default.target.backup
sudo systemctl set-default multi-user.target
sudo sync;reboot

# 如果想回图形界面启动
sudo systemctl set-default graphical.target   
sudo sync;reboot
```

### 连接服务器

```shell
# 改用mobaxterm 工具远程可正常显示，xshell 貌似连接后显示有问题，不清楚为啥会这样，联系麒麟他们要求打400，放弃。
# 使用免费版本 mobaxterm 即可，尽量避免使用盗版，容易染毒。
```

### 数据盘挂载

>根据自己的实际需要，选择适合自己的分区方式并挂载磁盘。

#### 分区常识
```shell
# 选择MBR 分区原则，单个挂载的数据盘大小在2T以内，包括2T；超过2T，请选择GPT 分区，不走LVM.

# 选择LVM 逻辑盘原则，单个挂载的数据盘大小在1T以内，包括1T。
# 磁盘小但为了方便扩容才需要LVM，根据自己需要考虑是否需要做。
# LVM 逻辑化多少会影响磁盘IO读写性能，不建议在数据库等对磁盘性能有要求等服务器上开启。
```

#### MBR分区，不做LVM

```shell
# 对数据盘进行MBR分区，创建文件系统，并挂载到/data目录，这里不做LVM
$ lsblk ## 查看自己当前磁盘
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0            11:0    1   4G  0 rom
nvme0n1       259:0    0  20G  0 disk
├─nvme0n1p1   259:1    0  19G  0 part
│ └─klas-root 253:0    0  19G  0 lvm  /
└─nvme0n1p2   259:2    0   1G  0 part /boot
nvme0n2       259:3    0  60G  0 disk

$ fdisk /dev/nvme0n2 # 对当前系统上空闲的磁盘，即尚未挂载的磁盘，进行分区(我这里是nvme0n2 空闲)

欢迎使用 fdisk (util-linux 2.35.2)。
更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

设备不包含可识别的分区表。
创建了一个磁盘标识符为 0x21366632 的新 DOS 磁盘标签。

命令(输入 m 获取帮助)：n  # 创建分区
分区类型
   p   主分区 (0 primary, 0 extended, 4 free)
   e   扩展分区 (逻辑分区容器)
选择 (默认 p)：p # 创建主分区
分区号 (1-4, 默认  1):
第一个扇区 (2048-125829119, 默认 2048):
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (2048-125829119, 默认 125829119):

创建了一个新分区 1，类型为“Linux”，大小为 60 GiB。

命令(输入 m 获取帮助)：t # 修改类型为lvm
已选择分区 1
Hex 代码(输入 L 列出所有代码)：8e
已将分区“Linux”的类型更改为“Linux LVM”。

命令(输入 m 获取帮助)：p # 打印当前分区结果
Disk /dev/nvme0n2：60 GiB，64424509440 字节，125829120 个扇区
磁盘型号：VMware Virtual NVMe Disk
单元：扇区 / 1 * 512 = 512 字节
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x21366632

设备           启动  起点      末尾      扇区 大小 Id 类型
/dev/nvme0n2p1       2048 125829119 125827072  60G 8e Linux LVM

命令(输入 m 获取帮助)：w  # 保存并退出分区操作
分区表已调整。
将调用 ioctl() 来重新读分区表。
正在同步磁盘。



$ partprobe # 刷新磁盘分区
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0            11:0    1   4G  0 rom
nvme0n1       259:0    0  20G  0 disk
├─nvme0n1p1   259:1    0  19G  0 part
│ └─klas-root 253:0    0  19G  0 lvm  /
└─nvme0n1p2   259:2    0   1G  0 part /boot
nvme0n2       259:3    0  60G  0 disk
└─nvme0n2p1   259:5    0  60G  0 part   ## 可以看到这里已经出来了

$ mkdir -p /data # 创建目录

$ mkfs.xfs /dev/nvme0n2p1 # 格式化磁盘，创建xfs文件系统
meta-data=/dev/nvme0n2p1         isize=512    agcount=4, agsize=3932096 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=15728384, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=7679, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0


$ blkid # 查看前面磁盘的UUID
/dev/nvme0n1p1: UUID="LptD0I-mVs7-bwch-PVJW-aqES-NyFz-3zQ37g" TYPE="LVM2_member" PARTUUID="b6332df0-01"
/dev/nvme0n1p2: UUID="c77d7864-2d1c-4aea-a2b2-0e18559761bf" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="b6332df0-02"
/dev/nvme0n2p1: UUID="b202bb87-a859-44d5-8e98-d716ae8437cc" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="21366632-01"
/dev/sr0: BLOCK_SIZE="2048" UUID="2021-08-09-17-54-20-00" LABEL="Kylin-Server-10" TYPE="iso9660"
/dev/mapper/klas-root: UUID="c795f9fd-6f62-4589-a859-e85bca1ea56a" BLOCK_SIZE="4096" TYPE="ext4"


$ vim /etc/fstab # 使用UUID 或目录给挂载配置写入系统配置
UUID=b202bb87-a859-44d5-8e98-d716ae8437cc /data xfs     defaults        0 0

$ mount -a # 刷新挂载

$ lsblk # 查看磁盘
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0            11:0    1   4G  0 rom
nvme0n1       259:0    0  20G  0 disk
├─nvme0n1p1   259:1    0  19G  0 part
│ └─klas-root 253:0    0  19G  0 lvm  /
└─nvme0n1p2   259:2    0   1G  0 part /boot
nvme0n2       259:3    0  60G  0 disk
└─nvme0n2p1   259:5    0  60G  0 part /data


$ df -Th /data # 确认文件系统
文件系统       类型  容量  已用  可用 已用% 挂载点
/dev/nvme0n2p1 xfs    60G  461M   60G    1% /data

```

#### MBR分区，做LVM
```shell
# 查看磁盘
lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0            11:0    1   4G  0 rom
nvme0n1       259:0    0  20G  0 disk
├─nvme0n1p1   259:1    0  19G  0 part
│ └─klas-root 253:0    0  19G  0 lvm  /
└─nvme0n1p2   259:2    0   1G  0 part /boot
nvme0n2       259:3    0  60G  0 disk
└─nvme0n2p1   259:4    0  60G  0 part /data
nvme0n3       259:5    0  10G  0 disk     ### 我这里有两块盘，计划给3号、4号盘做MBR分区+LVM，用4号盘通过LVM来给3号盘扩容
nvme0n4       259:6    0  10G  0 disk

# 先分区
$ fdisk /dev/nvme0n3

欢迎使用 fdisk (util-linux 2.35.2)。
更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

设备不包含可识别的分区表。
创建了一个磁盘标识符为 0xffc4fa17 的新 DOS 磁盘标签。

命令(输入 m 获取帮助)：n
分区类型
   p   主分区 (0 primary, 0 extended, 4 free)
   e   扩展分区 (逻辑分区容器)
选择 (默认 p)：

将使用默认回应 p。
分区号 (1-4, 默认  1):
第一个扇区 (2048-20971519, 默认 2048):
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (2048-20971519, 默认 20971519):

创建了一个新分区 1，类型为“Linux”，大小为 10 GiB。

命令(输入 m 获取帮助)：t
已选择分区 1
Hex 代码(输入 L 列出所有代码)：8e
已将分区“Linux”的类型更改为“Linux LVM”。

命令(输入 m 获取帮助)：w
分区表已调整。
将调用 ioctl() 来重新读分区表。
正在同步磁盘。

$ partprobe
Warning: 无法以读写方式打开 /dev/sr0 (只读文件系统)。/dev/sr0 已按照只读方式打开。

$ fdisk /dev/nvme0n4

欢迎使用 fdisk (util-linux 2.35.2)。
更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

设备不包含可识别的分区表。
创建了一个磁盘标识符为 0x9789bc6a 的新 DOS 磁盘标签。

命令(输入 m 获取帮助)：n
分区类型
   p   主分区 (0 primary, 0 extended, 4 free)
   e   扩展分区 (逻辑分区容器)
选择 (默认 p)：

将使用默认回应 p。
分区号 (1-4, 默认  1):
第一个扇区 (2048-20971519, 默认 2048):
最后一个扇区，+/-sectors 或 +size{K,M,G,T,P} (2048-20971519, 默认 20971519):

创建了一个新分区 1，类型为“Linux”，大小为 10 GiB。

命令(输入 m 获取帮助)：t
已选择分区 1
Hex 代码(输入 L 列出所有代码)：8e
已将分区“Linux”的类型更改为“Linux LVM”。

命令(输入 m 获取帮助)：w
分区表已调整。
将调用 ioctl() 来重新读分区表。
正在同步磁盘。

$ partprobe
Warning: 无法以读写方式打开 /dev/sr0 (只读文件系统)。/dev/sr0 已按照只读方式打开。

$ lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0            11:0    1   4G  0 rom
nvme0n1       259:0    0  20G  0 disk
├─nvme0n1p1   259:1    0  19G  0 part
│ └─klas-root 253:0    0  19G  0 lvm  /
└─nvme0n1p2   259:2    0   1G  0 part /boot
nvme0n2       259:3    0  60G  0 disk
└─nvme0n2p1   259:4    0  60G  0 part /data
nvme0n3       259:5    0  10G  0 disk
└─nvme0n3p1   259:8    0  10G  0 part
nvme0n4       259:6    0  10G  0 disk
└─nvme0n4p1   259:9    0  10G  0 part


# 开始创建LVM 逻辑盘
$ pvcreate /dev/nvme0n3p1  # 拿前面划出来的磁盘来创建PV
  Physical volume "/dev/nvme0n3p1" successfully created.

$ vgcreate vg_01 /dev/nvme0n3p1  # 拿PV 来创建VG 
  Volume group "vg_01" successfully created


$ lvcreate -l 100%FREE -n lv_01 vg_01 # 拿VG 来创建LV, 并将所有VG 空间给到LV
  Logical volume "lv_01" created.

$ mkfs.xfs /dev/vg_01/lv_01 # LV 磁盘格式化, 并创建xfs 文件系统
meta-data=/dev/vg_01/lv_01       isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0



$ mkdir -p /data1 # 创建挂载点

$ blkid | grep "lv_01"  # 查看UUID
/dev/mapper/vg_01-lv_01: UUID="e22e54f3-66f6-465f-99f2-05dab496f7db" BLOCK_SIZE="512" TYPE="xfs"

$ vim /etc/fstab   # 挂载配置写入系统配置
UUID=e22e54f3-66f6-465f-99f2-05dab496f7db /data1 xfs     defaults        0 0

$ mount -a # 刷新挂载

$ df -Th /data1 # 查看文件系统
文件系统                类型  容量  已用  可用 已用% 挂载点
/dev/mapper/vg_01-lv_01 xfs    10G  104M  9.9G    2% /data1

$ lsblk  # 查看磁盘
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0              11:0    1   4G  0 rom
nvme0n1         259:0    0  20G  0 disk
├─nvme0n1p1     259:1    0  19G  0 part
│ └─klas-root   253:0    0  19G  0 lvm  /
└─nvme0n1p2     259:2    0   1G  0 part /boot
nvme0n2         259:3    0  60G  0 disk
└─nvme0n2p1     259:4    0  60G  0 part /data
nvme0n3         259:5    0  10G  0 disk
└─nvme0n3p1     259:8    0  10G  0 part
  └─vg_01-lv_01 253:1    0  10G  0 lvm  /data1
nvme0n4         259:6    0  10G  0 disk
└─nvme0n4p1     259:9    0  10G  0 part



##### 接下来就是使用另外一块空闲磁盘，即前面的4号盘，来扩容3号LVM盘，这里不是必须，只是演示下在有这方面需要时应该如何操作

##### 这里需要强调一下，在磁盘扩容之前，如果之前磁盘上有十分重要的数据，请务必尽可能备份下，并不能保证100% 完全正常扩容。
##### 虽然因扩容而导致数据丢失的可能性微乎其微，但仍不是没有可能，请务必谨慎操作。

$ lsblk
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0              11:0    1   4G  0 rom
nvme0n1         259:0    0  20G  0 disk
├─nvme0n1p1     259:1    0  19G  0 part
│ └─klas-root   253:0    0  19G  0 lvm  /
└─nvme0n1p2     259:2    0   1G  0 part /boot
nvme0n2         259:3    0  60G  0 disk
└─nvme0n2p1     259:4    0  60G  0 part /data
nvme0n3         259:5    0  10G  0 disk
└─nvme0n3p1     259:8    0  10G  0 part
  └─vg_01-lv_01 253:1    0  10G  0 lvm  /data1
nvme0n4         259:6    0  10G  0 disk
└─nvme0n4p1     259:9    0  10G  0 part   ## 可以看到，这个是前面已经划好的磁盘


$ pvcreate /dev/nvme0n4p1 # 同样创建pv
  Physical volume "/dev/nvme0n4p1" successfully created.

$ vgextend vg_01 /dev/nvme0n4p1 # 拿上面的PV 给前面的VG 扩容
  Volume group "vg_01" successfully extended

$ lvextend -l +100%FREE /dev/vg_01/lv_01   # 让VG的所有空闲空间来给前面的LV 扩容
  Size of logical volume vg_01/lv_01 changed from <10.00 GiB (2559 extents) to 19.99 GiB (5118 extents).
  Logical volume vg_01/lv_01 successfully resized.


$ xfs_growfs /dev/vg_01/lv_01  # 刷新文件系统
meta-data=/dev/mapper/vg_01-lv_01 isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 2620416 to 5240832

$ df -Th /data1    # 查看文件系统，可以看到已经扩容成功
文件系统                类型  容量  已用  可用 已用% 挂载点
/dev/mapper/vg_01-lv_01 xfs    20G  176M   20G    1% /data1

$ lsblk  # 查看磁盘
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0              11:0    1   4G  0 rom
nvme0n1         259:0    0  20G  0 disk
├─nvme0n1p1     259:1    0  19G  0 part
│ └─klas-root   253:0    0  19G  0 lvm  /
└─nvme0n1p2     259:2    0   1G  0 part /boot
nvme0n2         259:3    0  60G  0 disk
└─nvme0n2p1     259:4    0  60G  0 part /data
nvme0n3         259:5    0  10G  0 disk
└─nvme0n3p1     259:8    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n4         259:6    0  10G  0 disk
└─nvme0n4p1     259:9    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1  # 可以看到都挂在一个/data1 目录下
```


#### GPT分区

```shell
$ lsblk   # 查看磁盘
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0              11:0    1   4G  0 rom
nvme0n1         259:0    0  20G  0 disk
├─nvme0n1p1     259:1    0  19G  0 part
│ └─klas-root   253:0    0  19G  0 lvm  /
└─nvme0n1p2     259:2    0   1G  0 part /boot
nvme0n2         259:3    0  60G  0 disk
└─nvme0n2p1     259:4    0  60G  0 part /data
nvme0n3         259:5    0  10G  0 disk
└─nvme0n3p1     259:8    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n4         259:6    0  10G  0 disk
└─nvme0n4p1     259:9    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n5         259:7    0  20G  0 disk     ## 拿这个盘做GPT 分区

$ parted /dev/nvme0n5  # 指定磁盘分区
GNU Parted 3.3
使用 /dev/nvme0n5
欢迎使用 GNU Parted！输入 'help' 来查看命令列表。
(parted) mklabel gpt  # 开始GPT分区
警告: 现有 /dev/nvme0n5 上的磁盘卷标将被销毁，而所有在这个磁盘上的数据将会丢失。您要继续吗？
是/Yes/否/No? yes  ## 可能会出现这个，若出现就输入yes
(parted) mkpart gpt-01 xfs 0% 100%   # 给当前磁盘所有空间划给一个盘符
(parted) quit  # 操作完成，退出
信息: 你可能需要 /etc/fstab。

$ partprobe  # 刷新磁盘
Warning: 无法以读写方式打开 /dev/sr0 (只读文件系统)。/dev/sr0 已按照只读方式打开。

$ lsblk  # 查看磁盘
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0              11:0    1   4G  0 rom
nvme0n1         259:0    0  20G  0 disk
├─nvme0n1p1     259:1    0  19G  0 part
│ └─klas-root   253:0    0  19G  0 lvm  /
└─nvme0n1p2     259:2    0   1G  0 part /boot
nvme0n2         259:3    0  60G  0 disk
└─nvme0n2p1     259:4    0  60G  0 part /data
nvme0n3         259:5    0  10G  0 disk
└─nvme0n3p1     259:8    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n4         259:6    0  10G  0 disk
└─nvme0n4p1     259:9    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n5         259:7    0  20G  0 disk
└─nvme0n5p1     259:10   0  20G  0 part   # 已经出来了


$ mkdir -p /data2  # 创建目录

$ mkfs.xfs /dev/nvme0n5p1  # 格式化磁盘，做xfs 文件系统
meta-data=/dev/nvme0n5p1         isize=512    agcount=4, agsize=1310592 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=5242368, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

$ blkid | grep gpt  # 查看UUID
/dev/nvme0n5p1: UUID="60d4c6f0-7fc3-4b88-b72c-65e79fe6fbfe" BLOCK_SIZE="512" TYPE="xfs" PARTLABEL="gpt-01" PARTUUID="766c0d85-f3e7-43cc-9949-eb677942a9c8"

$ vim /etc/fstab  # 挂载配置写入系统
UUID=60d4c6f0-7fc3-4b88-b72c-65e79fe6fbfe /data2 xfs     defaults       0 0

$ mount -a  # 刷新挂载

$ df -Th /data2  # 确认文件系统
文件系统       类型  容量  已用  可用 已用% 挂载点
/dev/nvme0n5p1 xfs    20G  176M   20G    1% /data2

$ lsblk  # 查看磁盘
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0              11:0    1   4G  0 rom
nvme0n1         259:0    0  20G  0 disk
├─nvme0n1p1     259:1    0  19G  0 part
│ └─klas-root   253:0    0  19G  0 lvm  /
└─nvme0n1p2     259:2    0   1G  0 part /boot
nvme0n2         259:3    0  60G  0 disk
└─nvme0n2p1     259:4    0  60G  0 part /data
nvme0n3         259:5    0  10G  0 disk
└─nvme0n3p1     259:8    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n4         259:6    0  10G  0 disk
└─nvme0n4p1     259:9    0  10G  0 part
  └─vg_01-lv_01 253:1    0  20G  0 lvm  /data1
nvme0n5         259:7    0  20G  0 disk
└─nvme0n5p1     259:10   0  20G  0 part /data2
```

### 清理家目录

```shell
# 这里是不用图形界面的情况，如果有需要用到图形界面，这里请忽略
$ ll
总用量 40
drwxr-xr-x 2 root root 4096  5月 30 13:12 公共
drwxr-xr-x 2 root root 4096  5月 30 13:12 模板
drwxr-xr-x 2 root root 4096  5月 30 13:12 视频
drwxr-xr-x 2 root root 4096  5月 30 13:12 图片
drwxr-xr-x 2 root root 4096  5月 30 13:12 文档
drwxr-xr-x 2 root root 4096  5月 30 13:12 下载
drwxr-xr-x 2 root root 4096  5月 30 13:12 音乐
drwxr-xr-x 2 root root 4096  5月 30 13:12 桌面
-rw------- 1 root root 2823  5月 30 13:11 anaconda-ks.cfg
-rw-r--r-- 1 root root 3159  5月 30 13:11 initial-setup-ks.cfg
$ rm * -rf
```



### 确认CPU架构

```shell
[root@KylinSp3 ~]# lscpu | grep Architecture
Architecture:                    aarch64
```



### 确认系统版本

```shell
[root@KylinSp3 ~]# cat /etc/.productinfo 
Kylin Linux Advanced Server
release V10 (SP3) /(Lance)-aarch64-Build23/20230324
```





### 服务器初始化

```shell
# 修改root 密码
[root@KylinSp3 ~]# sudo passwd root
# 切换到root
[root@KylinSp3 ~]# su -
# 修改root 可以直接登陆
[root@KylinSp3 ~]# sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config



# 不看到隐藏文件,不要加*，/，@等符号
[root@KylinSp3 ~]# sed -ri "s/alias ll='ls -alF'/alias ll='ls -l'/" ~/.bashrc
[root@KylinSp3 ~]# egrep "alias ll" ~/.bashrc
[root@KylinSp3 ~]# source ~/.bashrc


# 关闭防火墙，selinux 
[root@KylinSp3 ~]# getenforce 
Disabled
[root@KylinSp3 ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)



# 安装常用工具，多执行几次
[root@KylinSp3 ~]# yum install conntrack socat tree   net-tools genisoimage ntpdate jq sshpass -y 
[root@KylinSp3 ~]# yum install ansible  -y 
[root@KylinSp3 ~]# yum install python3-pip udev  python3 -y 


# 配置静态IP
[root@KylinSp3 ~]# ll /etc/sysconfig/network-scripts/ifcfg-enp4s0 
-rw-r--r-- 1 root root 248 Nov 21 16:38 /etc/sysconfig/network-scripts/ifcfg-enp4s0
[root@KylinSp3 ~]# vim /etc/sysconfig/network-scripts/ifcfg-enp4s0
[root@KylinSp3 ~]# cat /etc/sysconfig/network-scripts/ifcfg-enp4s0
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
NAME=enp4s0
UUID=1498733a-76a9-4a14-9565-0cd1f3bdb2c0
DEVICE=enp4s0
ONBOOT=yes
IPADDR=10.0.5.86
PREFIX=24
GATEWAY=10.0.5.1
DNS1=223.5.5.5


# 修改时区，同步时间
[root@KylinSp3 ~]# timedatectl set-timezone Asia/Shanghai
[root@KylinSp3 ~]# ntpdate -u ntp.aliyun.com
22 Nov 10:54:10 ntpdate[8640]: adjust time server 203.107.6.88 offset +0.009250 sec
[root@KylinSp3 ~]# echo "*/5 * * * * /usr/sbin/ntpdate -u ntp.aliyun.com >> /var/log/ntpdate.log 2>&1" | sudo tee /etc/cron.d/ntpdate-sync
*/5 * * * * /usr/sbin/ntpdate -u ntp.aliyun.com >> /var/log/ntpdate.log 2>&1


################################ 重启机器
sync;reboot
```





## Java环境

```shell

# 下载地址：
# https://www.oracle.com/java/technologies/javase/javase8u211-later-archive-downloads.html

# 上传文件到/root

# 解压释放
[root@KylinSp3 ~]# tar xf jdk-8u381-linux-aarch64.tar.gz -C /opt/
[root@KylinSp3 ~]# ll /opt/jdk1.8.0_381/
total 20704
drwxr-xr-x 2 10143 10143     4096 Jun 14 22:09 bin
-r--r--r-- 1 10143 10143     3244 Jun 14 22:09 COPYRIGHT
drwxr-xr-x 3 10143 10143      132 Jun 14 22:09 include
drwxr-xr-x 5 10143 10143      142 Jun 14 22:09 jre
drwxr-xr-x 3 10143 10143       17 Jun 14 22:09 legal
drwxr-xr-x 3 10143 10143      146 Jun 14 22:09 lib
-r--r--r-- 1 10143 10143       44 Jun 14 22:09 LICENSE
drwxr-xr-x 4 10143 10143       47 Jun 14 22:09 man
-r--r--r-- 1 10143 10143      159 Jun 14 22:09 README.html
-rw-r--r-- 1 10143 10143      185 Jun 14 22:09 release
-rw-r--r-- 1 10143 10143 21175606 Jun 14 22:09 src.zip
-r--r--r-- 1 10143 10143      190 Jun 14 22:09 THIRDPARTYLICENSEREADME.txt

# 配置环境变量
[root@KylinSp3 ~]# echo -e "export JAVA_HOME=/opt/jdk1.8.0_381" >> /etc/profile 
[root@KylinSp3 ~]# echo -e "export PATH=\$PATH:\$JAVA_HOME/bin" >> /etc/profile 
[root@KylinSp3 ~]# echo -e "export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar" >> /etc/profile 
[root@KylinSp3 ~]# echo -e "export JRE_HOME=\$JAVA_HOME/jre" >> /etc/profile 
# 确认配置
[root@KylinSp3 ~]# tail -4 /etc/profile
export JAVA_HOME=/opt/jdk1.8.0_381
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JRE_HOME=$JAVA_HOME/jre
[root@KylinSp3 ~]# source  /etc/profile



# 查看版本
[root@KylinSp3 ~]# java -version 
java version "1.8.0_381"
Java(TM) SE Runtime Environment (build 1.8.0_381-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.381-b09, mixed mode)
```



## Docker环境

```shell
# 官方文档地址：
# https://docs.docker.com/engine/install/binaries/

# 下载地址：
# https://download.docker.com/linux/static/stable/aarch64/

# 上传文件到/root/

# 解压安装包
[root@KylinSp3 ~]# tar xzvf docker-24.0.7.tgz 
docker/
docker/docker-init
docker/containerd-shim-runc-v2
docker/ctr
docker/docker-proxy
docker/dockerd
[root@KylinSp3 ~]# ll docker
total 173520
-rwxr-xr-x 1 1000 1000 37289984 Oct 26 17:10 containerd
-rwxr-xr-x 1 1000 1000 11862016 Oct 26 17:10 containerd-shim-runc-v2
-rwxr-xr-x 1 1000 1000 18219008 Oct 26 17:10 ctr
-rwxr-xr-x 1 1000 1000 33393056 Oct 26 17:10 docker
-rwxr-xr-x 1 1000 1000 59979448 Oct 26 17:10 dockerd
-rwxr-xr-x 1 1000 1000   535800 Oct 26 17:10 docker-init
-rwxr-xr-x 1 1000 1000  2046447 Oct 26 17:10 docker-proxy
-rwxr-xr-x 1 1000 1000 14350160 Oct 26 17:10 runc

# 将二进制文件放到PATH 目录下
[root@KylinSp3 ~]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/root/bin
[root@KylinSp3 ~]# \cp -rvf docker/* /usr/bin/
'docker/containerd' -> '/usr/bin/containerd'
'docker/containerd-shim-runc-v2' -> '/usr/bin/containerd-shim-runc-v2'
'docker/ctr' -> '/usr/bin/ctr'
'docker/docker' -> '/usr/bin/docker'
'docker/dockerd' -> '/usr/bin/dockerd'
'docker/docker-init' -> '/usr/bin/docker-init'
'docker/docker-proxy' -> '/usr/bin/docker-proxy'
'docker/runc' -> '/usr/bin/runc'

# 检查各个关键组件的版本
[root@KylinSp3 ~]# docker version 
Client:
 Version:           24.0.7
 API version:       1.43
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:06:50 2023
 OS/Arch:           linux/arm64
 Context:           default
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?              
[root@KylinSp3 ~]# runc --version 
runc version 1.1.9
commit: v1.1.9-0-gccaecfc
spec: 1.0.2-dev
go: go1.20.10
libseccomp: 2.5.1
[root@KylinSp3 ~]# containerd --version 
containerd github.com/containerd/containerd v1.7.6 091922f03c2762540fd057fba91260237ff86acb


########################################################################### 二进制安装compose

# 下载地址：
# https://github.com/docker/compose/releases/tag/v2.23.0

# 上传文件到/root

# 安装compose
[root@KylinSp3 ~]# \cp -rvf docker-compose-linux-aarch64  /usr/bin/docker-compose 
'docker-compose-linux-aarch64' -> '/usr/bin/docker-compose'
[root@KylinSp3 ~]# chmod +x /usr/bin/docker-compose

# 查看版本
[root@KylinSp3 ~]# docker-compose version 
Docker Compose version v2.23.0


########################################################################### 添加用户
# 官方文档地址：
# https://docs.docker.com/engine/install/linux-postinstall/

# docker 相关用户管理
[root@KylinSp3 ~]# groupadd docker
[root@KylinSp3 ~]# usermod -aG docker $USER
[root@KylinSp3 ~]# newgrp docker


########################################################################### 添加Containerd服务管理配置
# 参考配置地址
# https://github.com/moby/moby/tree/master/contrib/init/systemd

# 添加Containerd 配置文件
cat <<-EOF | tee  /usr/lib/systemd/system/containerd.service
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

# 服务管理
[root@KylinSp3 ~]# systemctl daemon-reload 
[root@KylinSp3 ~]# systemctl status containerd 
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: https://containerd.io
[root@KylinSp3 ~]# systemctl start  containerd 
[root@KylinSp3 ~]# systemctl enable  containerd 
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /usr/lib/systemd/system/containerd.service.
[root@KylinSp3 ~]# systemctl status containerd 
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-11-22 15:22:31 CST; 11s ago
     Docs: https://containerd.io
 Main PID: 10810 (containerd)
    Tasks: 10
   Memory: 13.0M
   CGroup: /system.slice/containerd.service
           └─10810 /usr/bin/containerd

Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463095454+08:00" level=info msg=serving... address=/run/containerd/containerd.sock.ttrpc
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463153194+08:00" level=info msg=serving... address=/run/containerd/containerd.sock
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463158344+08:00" level=info msg="Start subscribing containerd event"
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463275716+08:00" level=info msg="Start recovering state"
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463359777+08:00" level=info msg="Start event monitor"
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463382887+08:00" level=info msg="Start snapshots syncer"
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463395867+08:00" level=info msg="Start cni network conf syncer for default"
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463407527+08:00" level=info msg="Start streaming server"
Nov 22 15:22:31 KylinSp3.example.com containerd[10810]: time="2023-11-22T15:22:31.463468128+08:00" level=info msg="containerd successfully booted in 0.038387s"
Nov 22 15:22:31 KylinSp3.example.com systemd[1]: Started containerd container runtime.


########################################################################### 添加Docker服务管理配置
# 参考配置地址
# https://github.com/moby/moby/tree/master/contrib/init/systemd


# 下载地址：
# https://github.com/moby/moby/blob/master/contrib/init/systemd/docker.socket


# 上传文件到/root

cat <<-EOF | tee /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target docker.socket firewalld.service containerd.service time-set.target
Wants=network-online.target containerd.service
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutStartSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process
OOMScoreAdjust=-500

[Install]
WantedBy=multi-user.target
EOF

[root@KylinSp3 ~]# systemctl daemon-reload 

# 添加docker 核心配置
[root@KylinSp3 ~]# mkdir -p /etc/docker/ /data/docker


cat <<-EOF | tee /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://ustc-edu-cn.mirror.aliyuncs.com"],
  "debug": false,
  "insecure-registries": [
    "0.0.0.0/0"
  ],
  "ip-forward": true,
  "ipv6": false,
  "live-restore": true,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "100m",
    "max-file": "2"
  },
  "selinux-enabled": false,
  "experimental" : true,
  "storage-driver": "overlay2",
  "metrics-addr" : "0.0.0.0:9323",
  "data-root": "/data/docker",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

# Docker服务管理
[root@KylinSp3 ~]# \cp -rvf  docker.socket /usr/lib/systemd/system/
'docker.socket' -> '/usr/lib/systemd/system/docker.socket'
[root@KylinSp3 ~]# systemctl start docker 
Warning: The unit file, source configuration file or drop-ins of docker.service changed on disk. Run 'systemctl daemon-reload' to reload units.
[root@KylinSp3 ~]# systemctl daemon-reload 
[root@KylinSp3 ~]# systemctl status  docker 
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-11-22 15:36:43 CST; 12s ago
     Docs: https://docs.docker.com
 Main PID: 11251 (dockerd)
    Tasks: 12
   Memory: 29.1M
   CGroup: /system.slice/docker.service
           └─11251 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Nov 22 15:36:43 KylinSp3.example.com systemd[1]: Starting Docker Application Container Engine...
Nov 22 15:36:43 KylinSp3.example.com dockerd[11251]: time="2023-11-22T15:36:43.043012735+08:00" level=warning msg="Running experimental build"
Nov 22 15:36:43 KylinSp3.example.com systemd[1]: Started Docker Application Container Engine.
[root@KylinSp3 ~]# systemctl enable  docker 
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.

# 查看Docker 数据目录
[root@KylinSp3 ~]# ll /data/docker/
total 44
drwx--x--x 4 root root 4096 Nov 22 15:36 buildkit
drwx--x--- 2 root root 4096 Nov 22 15:36 containers
-rw------- 1 root root   36 Nov 22 15:36 engine-id
drwx------ 3 root root 4096 Nov 22 15:36 image
drwxr-x--- 3 root root 4096 Nov 22 15:36 network
drwx--x--- 3 root root 4096 Nov 22 15:36 overlay2
drwx------ 4 root root 4096 Nov 22 15:36 plugins
drwx------ 2 root root 4096 Nov 22 15:36 runtimes
drwx------ 2 root root 4096 Nov 22 15:36 swarm
drwx------ 2 root root 4096 Nov 22 15:36 tmp
drwx-----x 2 root root 4096 Nov 22 15:36 volumes
```

