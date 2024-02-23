## 部署Nacos 集群

>:warning: **这里部署和维护都是在/root 目录下，操作都是使用`root `用户，请留意。**



> :warning: **如果条件允许，建议各个集群都能独自使用自己的服务器资源，而非与其他集群共享，以免高负载时存在资源争抢导致故障的问题。**

### 1.1 使用方式

> :warning: **集群模式必须使用外置数据库做数据存储，内置数据库无法支持集群模式，目前官方仍然默认支持MYSQL,如果现场等保遇到安全漏洞问题，仍然需要同步处理。**



```shell
# 上传附件目录：nacos-cluster 到/root

# 查看文件
root@deepin-1:~# cd nacos-cluster
root@deepin-1:~/nacos-cluster# ls -lha 
total 390M
drwxr-xr-x 4 root root 4.0K Jan 22 16:10 .
drwx------ 8 root root 4.0K Jan 22 15:57 ..
drwxr-xr-x 2 root root 4.0K Jan 22 15:36 conf
-rw-r--r-- 1 root root 2.1K Jan 22 14:41 docker-compose.yaml
drwxr-xr-x 2 root root 4.0K Jan 22 15:42 env
-rw-r--r-- 1 root root   26 Jan 22 14:45 .env
-rw-r--r-- 1 root root   26 Jan 22 14:46 .env.example
-rw-r--r-- 1 root root 150M Jan 22 11:07 mysql.tar.gz
-rw-r--r-- 1 root root 224M Jan 22 11:07 nacos.tar.gz
-rw-r--r-- 1 root root  17M Jan 22 11:54 nginx.tar.gz

docker load -i nacos.tar.gz
docker load -i mysql.tar.gz
docker load -i nginx.tar.gz
cp .env.example .env
vim .env # 请结合自己的情况修改配置后再启动
docker-compose up -d
```



启动完成

```shell
root@deepin-1:~/nacos-cluster/nacos-mysql# docker-compose ps -a 
NAME          IMAGE                            COMMAND                  SERVICE   CREATED          STATUS                    PORTS
mysql         freemankevin/nacos-mysql:v8      "docker-entrypoint.s…"   mysql     44 minutes ago   Up 44 minutes (healthy)   0.0.0.0:3306->3306/tcp, 33060/tcp
nacos1        nacos/nacos-server:v2.3.0-slim   "bin/docker-startup.…"   nacos1    44 minutes ago   Up 44 minutes (healthy)   0.0.0.0:7848->7848/tcp, 0.0.0.0:8848->8848/tcp, 0.0.0.0:9868->9848/tcp, 0.0.0.0:9850->9849/tcp
nacos2        nacos/nacos-server:v2.3.0-slim   "bin/docker-startup.…"   nacos2    44 minutes ago   Up 44 minutes (healthy)   0.0.0.0:7849->7848/tcp, 0.0.0.0:8849->8848/tcp, 0.0.0.0:9869->9848/tcp, 0.0.0.0:9851->9849/tcp
nacos3        nacos/nacos-server:v2.3.0-slim   "bin/docker-startup.…"   nacos3    44 minutes ago   Up 44 minutes (healthy)   0.0.0.0:7850->7848/tcp, 0.0.0.0:8850->8848/tcp, 0.0.0.0:9870->9848/tcp, 0.0.0.0:9852->9849/tcp
nginx-nacos   nginx:1.25.3-alpine              "/docker-entrypoint.…"   nginx     44 minutes ago   Up 44 minutes (healthy)   80/tcp, 0.0.0.0:38848->8848/tcp
```





### 1.2 接口请求



```shell
# Service registration example
curl -X POST 'http://192.168.171.153:38848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080&username=nacos&accessToken=eyJhbGciOiJIUzM4NCJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcwNTkyMzkyNX0.9h7naJVDIyimqC873DbXONa9-BPvk6POUvvMuYbtIIw44Ukq83czbtpOj_2KzvSn'

# Service discovery example
curl -X GET 'http://192.168.171.153:38848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName&username=nacos&accessToken=eyJhbGciOiJIUzM4NCJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcwNTkyMzkyNX0.9h7naJVDIyimqC873DbXONa9-BPvk6POUvvMuYbtIIw44Ukq83czbtpOj_2KzvSn'

# Push configuration example
curl -X POST 'http://192.168.171.153:38848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld&username=nacos&accessToken=eyJhbGciOiJIUzM4NCJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcwNTkyMzkyNX0.9h7naJVDIyimqC873DbXONa9-BPvk6POUvvMuYbtIIw44Ukq83czbtpOj_2KzvSn'


# Get configuration examples
curl -X GET 'http://192.168.171.153:38848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&username=nacos&accessToken=eyJhbGciOiJIUzM4NCJ9.eyJzdWIiOiJuYWNvcyIsImV4cCI6MTcwNTkyMzkyNX0.9h7naJVDIyimqC873DbXONa9-BPvk6POUvvMuYbtIIw44Ukq83czbtpOj_2KzvSn'
```


