

## 部署Redis 集群

>:warning: **这里部署和维护都是在/root 目录下，操作都是使用`root `用户，请留意。**



> :warning: **如果条件允许，建议各个集群都能独自使用自己的服务器资源，而非与其他集群共享，以免高负载时存在资源争抢导致故障的问题。**



**集群模式：一主三从三哨兵**



这种`Redis`集群架构实际上就是把主从复制和哨兵模式结合使用的一个集群方案。更准确地说,它是一个:

**主从复制集群 + 哨兵模式集群**



主从复制集群部分:

- 1个主节点用于写
- 3个从节点与主节点主从同步实现备份



哨兵模式集群部分: 

- 3个哨兵节点监控主从状态
- 如果主节点故障,会自动选举从节点替换为新的主节点



所以这个集群同时实现了:

- 数据主从备份
- 自动故障转移
- 读写分离
- 高可用



通过结合主从复制和哨兵模式的优点,形成一个完整的`Redis`高可用集群架构。



### 1.1 使用方式



```bash
# 上传附件目录：redis-cluster 到/root

# 查看文件
root@deepin-1:~# cd redis-cluster
root@deepin-1:~/redis-cluster# ls -lha
total 11M
drwxr-xr-x 3 root root 4.0K Jan 18 18:27 .
drwx------ 5 root root 4.0K Jan 18 18:23 ..
drwxr-xr-x 2 root root 4.0K Jan 18 11:17 conf
-rw-r--r-- 1 root root 3.8K Jan 18 10:40 docker-compose.yml
-rw-r--r-- 1 root root  441 Jan 18 09:35 .env.example
-rw-r--r-- 1 root root  11M Jan 18 18:27 redis.tar.gz


docker load -i redis.tar.gz 
cp .env.example .env
vim .env # 请结合自己的情况修改配置后再启动
docker-compose up -d
```



启动完成

```shell
root@deepin-1:~/redis-cluster# docker-compose ps -a 
NAME                      IMAGE              COMMAND                  SERVICE                   CREATED          STATUS          PORTS
redis-server-master       redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-master       23 seconds ago   Up 21 seconds   0.0.0.0:6379->6379/tcp
redis-server-sentinel-1   redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-sentinel-1   23 seconds ago   Up 17 seconds   6379/tcp, 0.0.0.0:6383->26379/tcp
redis-server-sentinel-2   redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-sentinel-2   23 seconds ago   Up 16 seconds   6379/tcp, 0.0.0.0:6384->26379/tcp
redis-server-sentinel-3   redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-sentinel-3   23 seconds ago   Up 18 seconds   6379/tcp, 0.0.0.0:6385->26379/tcp
redis-server-slave-1      redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-slave-1      23 seconds ago   Up 15 seconds   0.0.0.0:6380->6379/tcp
redis-server-slave-2      redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-slave-2      23 seconds ago   Up 19 seconds   0.0.0.0:6381->6379/tcp
redis-server-slave-3      redis:6.2-alpine   "docker-entrypoint.s…"   redis-server-slave-3      23 seconds ago   Up 19 seconds   0.0.0.0:6382->6379/tcp
```

### 1.2 验证方式

#### 1.2.1 测试主节点状态



```shell
root@deepin-1:~/redis-cluster# docker exec -it redis-server-master redis-cli
127.0.0.1:6379> auth redis

127.0.0.1:6379> auth redis
OK
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:3
slave0:ip=10.10.10.3,port=6379,state=online,offset=15129,lag=0
slave1:ip=10.10.10.4,port=6379,state=online,offset=14994,lag=1
slave2:ip=10.10.10.2,port=6379,state=online,offset=14994,lag=1
master_failover_state:no-failover
master_replid:b5b020eda1ee0141e7066d6f2f59cd154c0fa145
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:15278
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:15278
```

#### 1.2.2 测试从节点状态

```shell
root@deepin-1:~/redis-cluster# docker exec -it redis-server-slave-1 redis-cli
127.0.0.1:6379> auth redis
OK
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> info replication
# Replication
role:slave
master_host:redis-server-master
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:26013
slave_repl_offset:26013
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:b5b020eda1ee0141e7066d6f2f59cd154c0fa145
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:26013
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:159
repl_backlog_histlen:25855
```



#### 1.2.3 测试哨兵节点状态

```
docker exec -it redis-server-sentinel-1 /bin/sh
/data # 
/data # redis-cli -p 26379 -h 10.10.10.5
10.10.10.5:26379> 
10.10.10.5:26379> 
10.10.10.5:26379> sentinel master mymaster
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "10.10.10.1"
 5) "port"
 6) "6379"
 7) "runid"
 8) "0b5c7ba11b4397c48e707ff15eb8b2f821dfd0ac"
 9) "flags"
10) "master"
11) "link-pending-commands"
12) "0"
13) "link-refcount"
14) "1"
15) "last-ping-sent"
16) "0"
17) "last-ok-ping-reply"
18) "240"
19) "last-ping-reply"
20) "240"
21) "down-after-milliseconds"
22) "5000"
23) "info-refresh"
24) "4750"
25) "role-reported"
26) "master"
27) "role-reported-time"
28) "185577"
29) "config-epoch"
30) "0"
31) "num-slaves"
32) "3"
33) "num-other-sentinels"
34) "2"
35) "quorum"
36) "2"
37) "failover-timeout"
38) "180000"
39) "parallel-syncs"
40) "1"
10.10.10.5:26379> 
```



#### 1.2.4 查看数据

```shell
root@deepin-1:~/redis-cluster# ls -lh /data/redis-*
/data/redis-master:
total 4.0K
-rw-r--r-- 1 deepin-anything-server deepin 176 Jan 18 18:29 dump.rdb

/data/redis-slave-1:
total 4.0K
-rw-r--r-- 1 deepin-anything-server deepin 176 Jan 18 18:29 dump.rdb

/data/redis-slave-2:
total 4.0K
-rw-r--r-- 1 deepin-anything-server deepin 175 Jan 18 18:29 dump.rdb

/data/redis-slave-3:
total 4.0K
-rw-r--r-- 1 deepin-anything-server deepin 175 Jan 18 18:29 dump.rdb
```

