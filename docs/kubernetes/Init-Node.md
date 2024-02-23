## 1 Refresh disk (optional)

```bash
# mount 
# /media/cdrom/
apt-cdrom add 
```



## 2 Add software source

```bash
# add internet repo
# 删除注释行（以#开头的行或者空行）
sed -i '/^#/d; /^$/d' /etc/apt/sources.list
# 注释掉原始行
sed -i 's/^deb cdrom:\[Debian GNU\/Linux.*$/#&/' /etc/apt/sources.list
# 删除安全行
sed -i '/^deb http:\/\/security.debian.org\/debian-security/d' /etc/apt/sources.list
sed -i '/^deb-src http:\/\/security.debian.org\/debian-security/d' /etc/apt/sources.list
# 添加新的行
sh -c 'echo "deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free" >> /etc/apt/sources.list'
sh -c 'echo "deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free" >> /etc/apt/sources.list'
sh -c 'echo "deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free" >> /etc/apt/sources.list'
sh -c 'echo "deb http://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free" >> /etc/apt/sources.list'

```



## 3 Update software

```bash
# update soft
apt update 
apt upgrade -y 
```



## 4 Installation tools

```bash
# use local repo to install pkg
apt update
apt install -y  \
apt-transport-https ca-certificates vim net-tools sudo telnet ufw iotop rsync unzip  \
iputils-ping  gawk sed  cron gnupg locales curl wget  nfs-common  sshpass tree software-properties-common

apt-get install nfs-common -y
```





## 5 Confirm system language

```bash
# change to en
locale-gen en_US.UTF-8
sed -i 's/LANG="zh_CN.UTF-8"/LANG="en_US.UTF-8"/' /etc/default/locale
sed -i 's/LANGUAGE="zh_CN:zh"/LANGUAGE="en_US:en"/' /etc/default/locale
sed -i 's/^zh_CN.UTF-8 UTF-8/#zh_CN.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen

cat <<-EOF | tee  /etc/default/locale
LC_ALL=en_US.utf8
LANG=en_US.utf8
LANGUAGE=en_US.utf8
EOF

locale-gen
# dpkg-reconfigure locales # 只选择 en_US.UTF-8，去掉 zh_CN.UTF-8 (若存在)


echo "export LANG=en_US.utf8" >> /etc/profile
echo "export LANGUAGE=en_US.utf8" >> /etc/profile
source /etc/profile
```



## 6 open remote

```bash
# export root ssh
sed -ri 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
service ssh restart
egrep '^PermitRootLogin' /etc/ssh/sshd_config
```



## 7 Modify display

```bash
# .bashrc

sed -i '/^# export LS_OPTIONS/s/^# //' ~/.bashrc
sed -i '/^# eval "\$(dircolors)"/s/^# //' ~/.bashrc
sed -i '/^# alias ls=/s/^# //' ~/.bashrc
sed -i '/^# alias ll=/s/^# //' ~/.bashrc
sed -i '/^# alias l=/s/^# //' ~/.bashrc

source ~/.bashrc
```



## 8 Modify static IP (optional)

> 若已配置请忽略

```bash
# static ip 
vim /etc/network/interfaces

$ cat  /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
#allow-hotplug ens32
#iface ens32 inet dhcp
auto ens32
iface ens32 inet static
    address 192.168.171.136
    netmask 255.255.255.0
    gateway 192.168.171.2
```



## 9 Close `swap`

```bash
# swapoff (if exited)
swapoff -a
sed -i '/swap/d' /etc/fstab
mount -a
```



## 10 Remove service

```bash
systemctl stop rpcbind
systemctl disable rpcbind
apt autoremove rpcbind -y
```



## 11 Restart the machine

```bash
# reboot
sync;reboot
```



