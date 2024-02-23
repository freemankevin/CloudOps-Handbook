

## 部署Nginx 集群

>:warning: **这里部署和维护都是在/root 目录下，操作都是使用`root `用户，请留意。**



> :warning: **如果条件允许，建议各个集群都能独自使用自己的服务器资源，而非与其他集群共享，以免高负载时存在资源争抢导致故障的问题。**



**集群模式：负载均衡集群**



当前 HAProxy 加上双 Nginx 的集群架构的优势和特点，可以用以下简单易懂的方式来表达：

**网络流量的智能分配（智能导航）**

想象一下，您有一个忙碌的商店，而且顾客络绎不绝。HAProxy 像是一个超级智能的迎宾，它可以迅速判断哪个售货员（在这里是 Nginx 服务器）当前比较空闲，并迅速将新顾客引导过去。这样，就确保了每个顾客都能尽快得到服务，没有一个售货员会过于繁忙而另外一些则闲着。

**服务的可靠性和持续性（不打烊的商店）**

如果其中一个售货员需要休息（比如一个 Nginx 服务器出现问题），HAProxy 这个迎宾会立即注意到，并且停止将顾客引导到他那里。相反，它会将顾客引导到其他能够提供服务的售货员那里。这就确保了商店（您的网站或应用）永远都是开着的，总有人在服务顾客，即使某个售货员需要临时离开。

**轻松处理高峰时段（繁忙时分也不怕）**

在促销活动或者节假日期间，客户的数量可能会激增。这个时候，有了 HAProxy 和两个 Nginx 服务器，这个系统就像是拥有了更多的售货员和一个能快速作出决策的迎宾，可以确保每个顾客即使在最繁忙的时候也能得到服务，而没有人需要排长队等待。

**安全保障（像保镖一样的安全）**

Nginx 不仅仅是售货员那么简单，它还像是商店的保安，能够检查每个进来的顾客（网络请求），确保它们是安全的，没有人是来捣乱的。如果有任何可疑的行为，Nginx 会阻止它们进入，保护您的商店（网站或应用）不受恶意攻击。

**灵活性和扩展性（随时随地扩大商店）**

随着您的生意越做越大，您可能需要更多的售货员和更大的商店空间。这个架构就允许您方便地增加更多的 Nginx 服务器（售货员）来处理更多的流量，或者升级现有的服务器以提供更快的服务。而且所有这些都可以在不打扰正在商店购物的顾客的情况下完成。

**简单的维护和管理（易于打理的商店）**

使用 Docker 和 Docker Compose，整个商店的管理变得极其简单。您可以很容易地对售货员进行培训（更新服务），重新布置商店（修改配置），或者在必要时重新开张（重启服务）。所有这些都可以通过简单的命令实现，不需要复杂的技术知识。

总而言之，HAProxy 加上双 Nginx 的集群就像是一个高效运转、永不休息、安全可靠，并且随时准备扩张的现代化商店，它能够确保每一位顾客都得到快速、安全的服务，同时也为店主（即您的业务）提供了轻松管理和维护的便利。





### 1.1 使用方式

```shell
# 上传附件目录：nginx-cluster 到/root

# 查看文件
root@deepin-1:~# cd nginx-cluster
root@deepin-1:~/nginx-cluster# ls -lha 
total 31M
drwxr-xr-x 5 root root 4.0K Jan 19 17:55 .
drwx------ 7 root root 4.0K Jan 19 17:55 ..
drwxr-xr-x 2 root root 4.0K Jan 19 16:50 certs
drwxr-xr-x 2 root root 4.0K Jan 19 17:39 conf
-rw-r--r-- 1 root root 1.9K Jan 19 17:33 docker-compose.yaml
-rw-r--r-- 1 root root   63 Jan 19 14:42 .env.example
-rw-r--r-- 1 root root  14M Jan 19 14:28 haproxy.tar.gz
drwxr-xr-x 2 root root 4.0K Jan 19 15:02 html
-rw-r--r-- 1 root root  17M Jan 19 11:03 nginx.tar.gz

# html 是你本地资源目录，需要上传你自己的静态资源
# conf/nginx.conf 是你的业务nginx 配置，需要你自己定义

docker load -i haproxy.tar.gz 
docker load -i nginx.tar.gz 
cp .env.example .env
vim .env # 请结合自己的情况修改配置后再启动
docker-compose up -d
```



启动完成

```shell
root@deepin-1:~/nginx-cluster# docker-compose ps -a
NAME                      IMAGE                 COMMAND                  SERVICE   CREATED         STATUS                   PORTS
nginx-cluster-haproxy-1   haproxy:alpine3.19    "docker-entrypoint.s…"   haproxy   3 minutes ago   Up 3 minutes (healthy)   0.0.0.0:8443->8443/tcp
nginx-cluster-nginx1-1    nginx:1.25.3-alpine   "/docker-entrypoint.…"   nginx1    3 minutes ago   Up 3 minutes (healthy)   80/tcp
nginx-cluster-nginx2-1    nginx:1.25.3-alpine   "/docker-entrypoint.…"   nginx2    3 minutes ago   Up 3 minutes (healthy)   80/tcp
```



### 1.2 制作任意域名证书

> 需要什么样域名的证书，自己通过修改这里的域名及文件名称即可



以要为域名`localserver.com`创建自签名的SSL证书举例，可以使用`openssl`命令。以下是一个命令行示例，它将生成自签名的SSL证书和私钥：



```shell
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout localserver.com.key -out localserver.com.crt -subj "/C=US/ST=State/L=City/O=Organization/CN=localserver.com"
```



合并证书和私钥

```shell
cat localserver.com.crt localserver.com.key > localserver.com.pem
```





这条命令执行了以下操作：

- `req`：这是对 OpenSSL 的请求，启动证书签名请求 (CSR) 管理。
- `-x509`：这会生成一个自签名的证书而非仅仅是一个 CSR。
- `-nodes`：这表示跳过保护私钥文件的密码。
- `-days 365`：这个证书将被设置为一年后过期。你可以根据需要改变这个值。
- `-newkey rsa:2048`：这会同时生成新的证书请求和 2048 位 RSA 密钥。
- `-keyout`：这指定了私钥文件的名称。
- `-out`：这指定了证书文件的名称。
- `-subj`：这允许你为你的证书指定所需的主题字段，避免了手动输入。



在上面的命令中，替换 `/C=US/ST=State/L=City/O=Organization/CN=localserver.com` 中的值，以反映你的组织和位置的正确信息。

执行这个命令后，你将会得到两个文件：`localserver.com.key`（你的私钥）和`localserver.com.crt`（你的公开证书）。这些文件可以在你的 HAProxy 和 Nginx 配置中使用，以启用 HTTPS。





### 1.3 查看日志



打开一个远程窗口，查看nginx1日志。

```shell
root@deepin-1:~/nginx-cluster# tail -100f /data/logs/nginx1/access.log 
192.168.128.2 - - [19/Jan/2024:17:41:00 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
192.168.128.2 - - [19/Jan/2024:17:41:07 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:10 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
192.168.128.2 - - [19/Jan/2024:17:41:15 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:15 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:15 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:20 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
```



打开另外一个远程窗口，查看nginx2日志。

```shell
root@deepin-1:~# tail -100f /data/logs/nginx2/access.log 
192.168.128.2 - - [19/Jan/2024:17:40:54 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
192.168.128.2 - - [19/Jan/2024:17:40:57 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:04 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
192.168.128.2 - - [19/Jan/2024:17:41:14 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
192.168.128.2 - - [19/Jan/2024:17:41:14 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:15 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:15 +0800] "GET / HTTP/1.1" 200 755 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
192.168.128.2 - - [19/Jan/2024:17:41:24 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-"
```



在浏览器中刷新页面，能发现nginx1 与nginx2 处理的请求是均等的。

也能通过haproxy 日志查看到

```shell
root@deepin-1:~/nginx-cluster# docker logs -f --tail 1000 nginx-cluster-haproxy-1
[NOTICE]   (1) : New worker (7) forked
[NOTICE]   (1) : Loading success.
192.168.171.1:2667 [19/Jan/2024:09:48:25.386] https_in/1: SSL handshake failure (error:0A000416:SSL routines::sslv3 alert certificate unknown)
192.168.171.1:2668 [19/Jan/2024:09:48:25.391] https_in~ http_servers/nginx1 0/0/5/0/5 200 1057 - - ---- 1/1/0/0/0 0/0 "GET https://192.168.171.153/ HTTP/2.0"
192.168.171.1:2668 [19/Jan/2024:09:48:25.415] https_in~ http_servers/nginx2 0/0/7/0/7 404 671 - - ---- 1/1/0/0/0 0/0 "GET https://192.168.171.153/favicon.ico HTTP/2.0"
192.168.171.1:2668 [19/Jan/2024:09:48:27.306] https_in~ http_servers/nginx1 0/0/0/0/0 200 1057 - - ---- 1/1/0/0/0 0/0 "GET https://192.168.171.153/ HTTP/2.0"
192.168.171.1:2668 [19/Jan/2024:09:48:27.632] https_in~ http_servers/nginx2 0/0/0/1/1 200 1057 - - ---- 1/1/0/0/0 0/0 "GET https://192.168.171.153/ HTTP/2.0"
192.168.171.1:2668 [19/Jan/2024:09:48:27.809] https_in~ http_servers/nginx1 0/0/0/0/0 200 1057 - - ---- 1/1/0/0/0 0/0 "GET https://192.168.171.153/ HTTP/2.0"
```

