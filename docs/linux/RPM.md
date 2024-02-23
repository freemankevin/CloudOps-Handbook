## 包免签名安装

```shell
rpm -ivh xxx.rpm --nodigest --nofiledigest
```

## 查看依赖关系

```shell
[root@mid yum]# rpm  -qR nmap-ncat
/bin/sh
/bin/sh
libc.so.6()(64bit)
libc.so.6(GLIBC_2.11)(64bit)
libc.so.6(GLIBC_2.14)(64bit)
libc.so.6(GLIBC_2.15)(64bit)
libc.so.6(GLIBC_2.2.5)(64bit)
libc.so.6(GLIBC_2.3)(64bit)
libc.so.6(GLIBC_2.3.2)(64bit)
libc.so.6(GLIBC_2.3.4)(64bit)
libc.so.6(GLIBC_2.4)(64bit)
libc.so.6(GLIBC_2.7)(64bit)
libc.so.6(GLIBC_2.8)(64bit)
libcrypto.so.10()(64bit)
libcrypto.so.10(libcrypto.so.10)(64bit)
libdl.so.2()(64bit)
libdl.so.2(GLIBC_2.2.5)(64bit)
libm.so.6()(64bit)
libm.so.6(GLIBC_2.2.5)(64bit)
libpcap.so.1()(64bit)
libssl.so.10()(64bit)
libssl.so.10(libssl.so.10)(64bit)
rpmlib(CompressedFileNames) <= 3.0.4-1
rpmlib(FileDigests) <= 4.6.0-1
rpmlib(PayloadFilesHavePrefix) <= 4.0-1
rtld(GNU_HASH)
rpmlib(PayloadIsXz) <= 5.2-1
```


## 包卸载

```shell
# 查包
rpm -qa | grep -i 服务名
# 卸载包
rpm -e 服务名对应的包名 --nodeps
rpm -e --nodeps java-1.6.0-openjdk-1.6.0.0-1.66.1.13.0.el6.i686
```

## 包制作

```shell
# 描述阶段

[root@localhost ~]# rpm -qpi grafana-8.0.6-1.x86_64.rpm
warning: grafana-8.0.6-1.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 24098cb6: NOKEY
Name        : grafana
Version     : 8.0.6  # 不能使用横线
Release     : 1      # 第几次制作，发行号
Architecture: x86_64 
Install Date: (not installed)
Group       : default  # 对应当前rpm制作所在的系统的所拥有的组名
Size        : 182016648
License     : "Apache 2.0"
Signature   : RSA/SHA256, Thu 15 Jul 2021 06:57:12 PM CST, Key ID 8c8c34c524098cb6
Source RPM  : grafana-8.0.6-1.src.rpm
Build Date  : Thu 15 Jul 2021 06:56:44 PM CST
Build Host  : 789e9d678c8c
Relocations : /
Packager    : contact@grafana.com  # 制作者<制作者邮件地址>
Vendor      : Grafana              # 制作者所属公司或提供商
URL         : https://grafana.com    # 下载链接
Summary     : Grafana
Description :
Grafana

# rpm 组

[root@localhost ~]# cat /usr/share/doc/rpm-4.11.3/GROUPS
Amusements/Games
Amusements/Graphics
Applications/Archiving
Applications/Communications
Applications/Databases
Applications/Editors
Applications/Emulators
Applications/Engineering
Applications/File
Applications/Internet
Applications/Multimedia
Applications/Productivity
Applications/Publishing
Applications/System
Applications/Text
Development/Debuggers
Development/Languages
Development/Libraries
Development/System
Development/Tools
Documentation
System Environment/Base
System Environment/Daemons
System Environment/Kernel
System Environment/Libraries
System Environment/Shells
User Interface/Desktops
User Interface/X
User Interface/X Hardware Support

# rpm tmp目录

[root@localhost ~]# rpmbuild --showrc|grep _tmppath
-14: _tmppath   %{_var}/tmp

# 准备阶段

%prep # 声明准备阶段
%setup # 宏动作，解压释放，准备环境等

# 编译阶段

%build
./configure \
	--etcdir="%{_sysconfir}"\   # %{变量}  不区分大小写
  --mandir="%{_mandir}"\
  ...
%{_make} %{?_smp_mflags} # 判断后再make

# 安装阶段

%install
%{_rm} -rf %{buildroot}
%{_make} install DESTEDIR="%{buildroot}"
%find_lang  %{name}

# 脚本阶段

%pre  #安装前
%post #安装后
%preun  # 卸载前
%postun # 卸载后

# 清理阶段

%clean
%{_rm} -rf %{buildroot}

# 文件阶段

%file

# 日志阶段

rpm 构建参数

-ba  生成rpm/tar.gz
-bb  生成rpm
-bc  构建至%build
-bp  构建至%pre
-bi  构建至%install
-bl  检查%file 阶段的文件与%{buildroot} 目录中文件的一一对应关系,正常应完全对应
-bs  生成src.rpm
```

## 从源码制作rpm 

[https://www.cnblogs.com/michael-xiang/p/10480809.html](https://www.cnblogs.com/michael-xiang/p/10480809.html)
[https://blog.csdn.net/u012373815/article/details/73257754](https://blog.csdn.net/u012373815/article/details/73257754)


```shell
yum -y install rpm-build rpm-devel rpmdevtools


useradd rpmuser
su - rpmuser
# rpmdev-setuptree
mkdir ~/rpmbuild
cd ~/rpmbuild
mkdir -pv {BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}


[rpmuser@localhost rpmbuild]$ tree
.
├── BUILD
├── RPMS
├── SOURCES
├── SPECS
└── SRPMS

5 directories, 0 files
[rpmuser@localhost rpmbuild]$

cp ~/grafana-8.1.0.tar.gz /home/rpmuser/rpmbuild/SOURCES
cd /home/rpmuser/rpmbuild/SPECS
rpmdev-newspec -o grafana.spec
rpmbuild -bb /home/rpmuser/rpmbuild/SPECS/grafana.spec

rpmbuild  
-ba 既生成src.rpm又生成二进制rpm 
-bs 只生成src的rpm 
-bb 只生二进制的rpm 
-bp 执行到pre 
-bc 执行到 build段 
-bi 执行install段 
-bl 检测有文件没包含 

```


## 忽略依赖安装

```shell
# 这个命令可以使rpm 安装时不检查依赖关系
rpm -i software-2.3.4.rpm --nodeps

# 这个命令可以强制安装rpm包
rpm -ivh software-2.3.4.rpm --force

rpm  -ivh --nodeps  --force
```

## 包下载网站

- [http://rpmfind.net](http://rpmfind.net)
- https://pkgs.org/download/rpm


## 查看已安装完成的包
```shell
rpm -qa | grep jdk
```

## 升级版本
```shell
#升级
rpm -Uvh grafana-6.4.0-1.x86_64.rpm 
```
