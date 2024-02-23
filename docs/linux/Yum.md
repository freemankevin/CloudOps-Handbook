

## 查看所有版本

```shell
yum list docker-ce --showduplicate
```

## 安装htop

```shell
dnf -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm \
    https://dl.fedoraproject.org/pub/epel/epel-next-release-latest-9.noarch.rpm

dnf install htop -y
```

## 列出所有版本

```shell
yum list docker-ce --showduplicates | sort -r
yum list java-1.10 --showduplicates | sort -r
```



## 安装指定版本包

```shell
yum list kubectl  --showduplicates | sort -r
yum -y install kubectl-1.20.6
```

## 整合离线包

[https://www.1024sou.com/article/404581.html](https://www.1024sou.com/article/404581.html)

```shell
yum -y install yum-utils
yumdownloader --resolve --destdir=/tmp docker
```



## 查看已安装

```shell
$ yum list installed|grep python3
$ yum list installed wget
```



## 本地仓库搭建

[https://blog.csdn.net/qq_29974229/article/details/103827369](https://blog.csdn.net/qq_29974229/article/details/103827369)





### 添加仓库

```shell
# 安装建库所需工具
yum install -y yum-utils device-mapper-persistent-data lvm2 createrepo wget  curl epel-release


# 下载仓库
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo
```



### 手动同步

```shell
# centos 7
reposync -r base             -p /data/yum  
reposync -r epel             -p /data/yum
reposync -r extras           -p /data/yum
reposync -r updates          -p /data/yum
reposync -r kubernetes       -p /data/yum  
reposync -r docker-ce-stable -p /data/yum
```



### 创建本地源

```shell
# 创建 reopdata仓库，生成仓库信息
createrepo -v /data/yum/epel
createrepo -v /data/yum/base
createrepo -v /data/yum/extras
createrepo -v /data/yum/updates
createrepo -v /data/yum/kubernetes
createrepo -v /data/yum/docker-ce-stable
```

## 脚本同步

```shell
#!/bin/bash
##specify all local repositories in a single variable
LOCAL_REPOS="base extras updates epel centosplus docker-ce-stable kubernetes erlang-solutions"
##local yum path
DOWNLOAD_PATH=/data/yum/
##a loop to update repos one at a time
for REPO in ${LOCAL_REPOS}; do

if [[ $REPO = 'kubernetes' || $REPO = 'docker-ce-stable' || $REPO = 'erlang-solutions' ]];then
    reposync -r $REPO -p $DOWNLOAD_PATH
else
    reposync -g -l -d -m -n --download-metadata -r $REPO  -p $DOWNLOAD_PATH
fi

if [[ $REPO = 'base' || $REPO = 'epel' ]]; then
    createrepo -g $DOWNLOAD_PATH/$REPO/comps.xml $DOWNLOAD_PATH/$REPO/
else
    createrepo $DOWNLOAD_PATH/$REPO/
fi
done
```


### 内网共享


```shell
yum install nginx -y

cp -rf /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
grep -vE "#|^$" /etc/nginx/nginx.conf.bak >/etc/nginx/nginx.conf.bak

cat <<eof >/etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events {
    worker_connections 1024;
}
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    include /etc/nginx/conf.d/*.conf;
    server {
        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name yum;
        root /data/yum;
        location / {
        }
        error_page 404 /404.html;
        location = /404.html {
        }
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
}
eof



systemctl restart nginx
systemctl enable nginx
ss -ntulp |grep 80
```



### 本地仓库使用

```shell
# 将原repo文件备份，替换为新的即可
cd /etc/yum.repos.d
mkdir -p repo.bak
mv *.repo repo.bak/


# 添加新的repo文件
cat <<eof > own.repo
[base]
name=CentOS-$releasever - Base
baseurl=file:///data/yum/base
enabled=1
gpgcheck=0

[extras]
name=CentOS-$releasever - Base-ex
baseurl=file:///data/yum/extras
enabled=1
gpgcheck=0

[updates]
name=CentOS-$releasever - Base-updates
baseurl=file:///data/yum/updates
enabled=1
gpgcheck=0

[epel]
name=epel
baseurl=file:///data/yum/epel
enabled=1
gpgcheck=0

[centosplus]
name=centosplus
baseurl=file:///data/yum/centosplus
enabled=1
gpgcheck=0

[docker]
name=docker-ce
baseurl=file:///data/yum/docker-ce-stable
enabled=1
gpgcheck=0

[k8s]
name=kubernetes
baseurl=file:///data/yum/kubernetes
enabled=1
gpgcheck=0
eof
```





### 远程仓库使用

```shell
# 将原repo文件备份，替换为新的即可
cd /etc/yum.repos.d
mkdir -p repo.bak
mv *.repo repo.bak/


# 添加新的repo文件
cat <<eof > own.repo
[base]
name=CentOS-$releasever - Base
baseurl=http://10.0.1.2/base
enabled=1
gpgcheck=0

[extras]
name=CentOS-$releasever - Base-ex
baseurl=http://10.0.1.2/extras
enabled=1
gpgcheck=0

[updates]
name=CentOS-$releasever - Base-updates
baseurl=http://10.0.1.2/updates
enabled=1
gpgcheck=0

[epel]
name=epel
baseurl=http://10.0.1.2/epel
enabled=1
gpgcheck=0

[docker]
name=docker-ce
baseurl=http://10.0.1.2/docker-ce-stable
enabled=1
gpgcheck=0

[k8s]
name=kubernetes
baseurl=http://10.0.1.2/kubernetes
enabled=1
gpgcheck=0
eof
```



## 禁用注册

>`yum ：  This system is not registered with an entitlement server. You can use subscription-manager to`

```shell
[root@Oradb1 pluginconf.d]# vim /etc/yum/pluginconf.d/subscription-manager.conf
[main]
enabled=0           #将它禁用掉
```



## 软件包安装冲突

> 问题现象：--> 处理 avahi-libs-0.6.31-13.el7.x86_64 与 avahi > 0.6.31-13.el7 的冲突
> --> 处理 avahi-ui-gtk3-0.6.31-13.el7.x86_64 与 avahi > 0.6.31-13.el7 的冲突
> --> 解决依赖关系完成
> 错误：软件包：avahi-libs-0.6.31-13.el7.x86_64 (@anaconda)
>           需要：avahi = 0.6.31-13.el7
>           正在删除: avahi-0.6.31-13.el7.x86_64 (@anaconda)
>               avahi = 0.6.31-13.el7
>           更新，由: avahi-0.6.31-14.el7.x86_64 (base)
>              avahi = 0.6.31-14.el7
> 错误：avahi-libs conflicts with avahi-0.6.31-14.el7.x86_64
> 错误：avahi-autoipd conflicts with avahi-0.6.31-14.el7.x86_64
> 错误：avahi-glib conflicts with avahi-0.6.31-14.el7.x86_64
> 错误：avahi-ui-gtk3 conflicts with avahi-0.6.31-14.el7.x86_64
> 错误：avahi-gobject conflicts with avahi-0.6.31-14.el7.x86_64
> 您可以尝试添加 --skip-broken 选项来解决该问题
> ** 发现 426 个已存在的 RPM 数据库问题， 'yum check' 输出如下：
> 1:NetworkManager-1.0.0-14.git20150121.b4ea599c.el7.x86_64 是 1:NetworkManager-0.9.9.1-29.git20140326.4dba720.el7_0.x86_64 的副本
> 1:NetworkManager-glib-1.0.0-14.git20150121.b4ea599c.el7.x86_64 是 1:NetworkManager-glib-0.9.9.1-29.git20140326.4dba720.el7_0.x86_64 的副本
> 1:NetworkManager-libnm-1.0.0-14.git20150121.b4ea599c.el7.x86_64 有已安装冲突 NetworkManager-glib < ('1', '1.0.0', '1'): 1:NetworkManager-glib-0.9.9.1-29.git20140326.4dba720.el7_0.x86_64
> 1:NetworkManager-tui-1.0.0-14.git20150121.b4ea599c.el7.x86_64 是 1:NetworkManager-tui-0.9.9.1-29.git20140326.4dba720.el7_0.x86_64 的副本
> ...



```shell
yum install yum-utils  
yum-complete-transaction --cleanup-only  
#清除可能存在的重复包
package-cleanup --dupes  
#清除可能存在的损坏包
package-cleanup --problems  
#清除重复包的老版本：
package-cleanup --cleandupes**

# 或者直接 忽略依赖强制删除上面提示"Removing"的rpm包
rpm -e --nodeps systemd-219-42.el7_4.4.x86_64
```
