## 1 系统初始化

### 1.1 修改root密码

```shell
# 只有普通用户属于admin 管理组即可有权限
sudo passwd root
```

### 1.2 sudo 免密码

```shell
# 切到root
$ su -
# 修改权限
$ chmod 740 /etc/sudoers
$ vim /etc/sudoers
#%sudo  ALL=(ALL:ALL) ALL # 将原有这行注释了，改成下面这种
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
# 权限改回去
$ chmod 0440 /etc/sudoers 
```

### 1.3 修改端口范围

```shell
[root@localhost ~]# cat /proc/sys/net/ipv4/ip_local_port_range
32768   61000
[root@localhost ~]#  echo 1024 65535 > /proc/sys/net/ipv4/ip_local_port_range
```

### 1.4 常用工具

```shell
sudo apt -y  install ssh vim htop tree net-tools  lsof lrzsz   curl   ebtables conntrack socat wget   telnet chrony rsync tcpdump htop
xfsprogs
```

### 1.5 切换源

```shell
# deb行是相对于二进制包的，您可以使用apt.
# deb-src行是相对于源包（由 下载apt-get source $package）和下一次编译的。
# 仅当您想自己编译某个包或检查源代码是否存在错误时，才需要源包。普通用户不需要包含这样的存储库。

$ vim /etc/apt/sources.list

## 清华源
# 20.04 LTS

# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse


### 中科大
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse

### 网易
deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse

### 阿里云
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```



```shell
# 或者
# 切换软件源
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo mv /etc/apt/sources.list.d/original.list{,.bak}
sudo tee /etc/apt/sources.list <<-'EOF'
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
EOF

### arm64 ubuntu 22.04 lts

# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse

# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
# # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse

deb http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted universe multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
# # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
```



### 1.6 静态IP配置

[https://ld246.com/article/1593929878472](https://ld246.com/article/1593929878472)

```shell

sudo vi /etc/netplan/00-installer-config.yaml
###################################################
network:
  ethernets:
    ens160:     # 网卡名
      addresses: [10.1.6.248/19]    # 静态ip/掩码
      dhcp4: no    # 关闭DHCP
      optional: true
      gateway4: 10.1.0.254    # 网关
      nameservers:
         addresses: [114.114.114.114,8.8.8.8]   
  version: 2
###################################################
sudo netplan apply

# 或者重启机器生效
sync;reboot
```

### 1.7 ARM-22.04-源

```shell
## 默认
cat /etc/apt/sources.list
# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb http://ports.ubuntu.com/ubuntu-ports/ focal main restricted
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal main restricted

## Major bug fix updates produced after the final release of the
## distribution.
deb http://ports.ubuntu.com/ubuntu-ports/ focal-updates main restricted
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-updates main restricted

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team. Also, please note that software in universe WILL NOT receive any
## review or updates from the Ubuntu security team.
deb http://ports.ubuntu.com/ubuntu-ports/ focal universe
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal universe
deb http://ports.ubuntu.com/ubuntu-ports/ focal-updates universe
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-updates universe

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team, and may not be under a free licence. Please satisfy yourself as to
## your rights to use the software. Also, please note that software in
## multiverse WILL NOT receive any review or updates from the Ubuntu
## security team.
deb http://ports.ubuntu.com/ubuntu-ports/ focal multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ focal-updates multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-updates multiverse

## N.B. software from this repository may not have been tested as
## extensively as that contained in the main release, although it includes
## newer versions of some applications which may provide useful features.
## Also, please note that software in backports WILL NOT receive any review
## or updates from the Ubuntu security team.
deb http://ports.ubuntu.com/ubuntu-ports/ focal-backports main restricted universe multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-backports main restricted universe multiverse

## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu focal partner
# deb-src http://archive.canonical.com/ubuntu focal partner

deb http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted
deb http://ports.ubuntu.com/ubuntu-ports/ focal-security universe
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-security universe
deb http://ports.ubuntu.com/ubuntu-ports/ focal-security multiverse
# deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-security multiverse

## 默认-精简
deb http://ports.ubuntu.com/ubuntu-ports/ focal main restricted
deb http://ports.ubuntu.com/ubuntu-ports/ focal-updates main restricted
deb http://ports.ubuntu.com/ubuntu-ports/ focal universe
deb http://ports.ubuntu.com/ubuntu-ports/ focal-updates universe
deb http://ports.ubuntu.com/ubuntu-ports/ focal multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ focal-updates multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ focal-backports main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted
deb http://ports.ubuntu.com/ubuntu-ports/ focal-security universe
deb http://ports.ubuntu.com/ubuntu-ports/ focal-security multiverse


## 安装工具
apt install -y apt-transport-https ca-certificates 
apt update


## 清华源
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
```

### 1.8 启用root用户

```shell
#  root远程登录
$ sudo vi /etc/ssh/sshd_config
PermitRootLogin  yes
# 或者
sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

$ sudo  systemctl  restart  ssh


# 改密码
sudo passwd root
```

### 1.9 使用nfs

```shell
sudo  apt update  -y
sudo  apt upgrade -y 

sudo apt-get install nfs-common -y  # 经测试，可用
sudo apt-get install nfs-utils -y # 经测试，不可用
```



### 1.10 修改显示

```shell
# 不看到隐藏文件,不要加*，/，@等符号
sed -ri "s/alias ll='ls -alF'/alias ll='ls -l'/" ~/.bashrc
egrep "alias ll" ~/.bashrc
source ~/.bashrc
```



### 1.11 关闭防火墙

```shell

# 关闭防火墙
ufw disable
ufw status
```



### 1.12 软件更新

```shell
apt clean
apt update
apt upgrade
apt dist-upgrade
```



### 1.13 安装软件

```shell
sudo apt -y install conntrack socat tree   net-tools genisoimage ntpdate jq sshpass 
sudo apt -y install ansible 
sudo apt -y install python3-pip udev netplan.io python3.10-venv 
sudo netplan apply
```



### 1.14 同步时区

```shell
timedatectl set-timezone Asia/Shanghai
ntpdate -u ntp.aliyun.com
echo "*/5 * * * * /usr/sbin/ntpdate -u ntp.aliyun.com >> /var/log/ntpdate.log 2>&1" | sudo tee /etc/cron.d/ntpdate-sync
```



### 1.15 根目录扩容

```shell
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h /dev/ubuntu-vg/ubuntu-lv
```



## 2 有坑

### 2.1 驱动
```
1.蓝牙驱动之类的硬件兼容性支持可能不如windows
```

### 2.2 服务

```
1. smb 服务尽量不要用，这里时不时会有服务异常
2. 开机自启的rc.d 脚本，很多时候没有生效
3. 软件包的源太大，完全离线下来不太可能
```
