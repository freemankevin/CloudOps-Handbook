## 部署Minio集群

>:warning: **这里部署和维护都是在/root 目录下，操作都是使用`root `用户，请留意。**



> :warning: **如果条件允许，建议各个集群都能独自使用自己的服务器资源，而非与其他集群共享，以免高负载时存在资源争抢导致故障的问题。**



**集群模式：多实例分布式集群**

这里的方案是一个分布式的 MinIO 对象存储集群，采用了多实例部署模式。这种部署模式有以下亮点优势：

1. **高可用性**：通过部署多个 MinIO 实例，可以实现高可用性。如果一个实例出现故障，其他实例仍然可以继续提供服务，确保系统的稳定性和可用性。

2. **负载均衡**：使用 Nginx 反向代理实现负载均衡，将客户端请求分发到多个 MinIO 实例上。这有助于提高性能并避免单点故障。

3. **数据冗余**：通过将数据存储在多个数据卷上，实现了数据冗余和容错性。即使某个数据卷出现问题，数据仍然可以从其他数据卷中恢复。

4. **可扩展性**：采用了容器化部署，可以相对容易地扩展集群规模，以满足不断增长的存储需求。

总的来说，这种部署模式能够提供高可用性、负载均衡、数据冗余和可扩展性，适用于需要大规模存储和高性能的应用场景，同时也能够应对硬件故障和数据丢失的风险。



### 1.1 使用方式

```shell
# 上传附件目录：minio-cluster 到/root

# 查看文件
root@deepin-1:~/minio-cluster# ls -lha
total 67M
drwxr-xr-x 2 root root 4.0K Jan 19 12:01 .
drwx------ 6 root root 4.0K Jan 19 12:01 ..
-rw-r--r-- 1 root root 1.5K Jan 19 11:59 docker-compose.yaml
-rw-r--r-- 1 root root  183 Jan 18 09:35 .env.example
-rw-r--r-- 1 root root  50M Jan 19 10:59 minio.tar.gz
-rw-r--r-- 1 root root 3.0K Jan 19 10:56 nginx.conf
-rw-r--r-- 1 root root  17M Jan 19 11:03 nginx.tar.gz

docker load -i minio.tar.gz 
cp .env.example .env
vim .env # 请结合自己的情况修改配置后再启动
docker-compose up -d
```



启动完成

```shell
root@deepin-1:~/minio-cluster# docker-compose ps -a
NAME                     IMAGE                                              COMMAND                  SERVICE   CREATED         STATUS                   PORTS
minio-cluster-minio1-1   quay.io/minio/minio:RELEASE.2024-01-16T16-07-38Z   "/usr/bin/docker-ent…"   minio1    6 minutes ago   Up 6 minutes (healthy)   9000-9001/tcp
minio-cluster-minio2-1   quay.io/minio/minio:RELEASE.2024-01-16T16-07-38Z   "/usr/bin/docker-ent…"   minio2    6 minutes ago   Up 6 minutes (healthy)   9000-9001/tcp
minio-cluster-minio3-1   quay.io/minio/minio:RELEASE.2024-01-16T16-07-38Z   "/usr/bin/docker-ent…"   minio3    6 minutes ago   Up 6 minutes (healthy)   9000-9001/tcp
minio-cluster-minio4-1   quay.io/minio/minio:RELEASE.2024-01-16T16-07-38Z   "/usr/bin/docker-ent…"   minio4    6 minutes ago   Up 6 minutes (healthy)   9000-9001/tcp
minio-cluster-nginx-1    nginx:1.25.3-alpine                                "/docker-entrypoint.…"   nginx     6 minutes ago   Up 6 minutes             80/tcp, 0.0.0.0:9000-9001->9000-9001/tcp
```



### 1.2 部署模板

这里因为本地系统的磁盘数量不同，而有不同的部署方式，参考这里模板修改为你自己符合自己实际需要的即可。

> :warning:  生产环境建议至少2块磁盘，对性能如果有要求，请至少使用[SSD磁盘](https://plantegg.github.io/2020/01/25/ssd%20san%E5%92%8Csas%20%E7%A3%81%E7%9B%98%E6%80%A7%E8%83%BD%E6%AF%94%E8%BE%83/), 更高要求可以使用[NVMe SSD](https://www.kingston.com.cn/cn/ssd/what-is-nvme-ssd-technology)。



**如何查看磁盘？**

```shell
# 如这里sda 是系统盘,sdb是数据盘
# 你所需要的就是要确保自己系统有多块sdb这样的数据盘，这里是做了LVM 方便后期扩展磁盘的。
root@debian:~# lsblk 
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  100G  0 disk   
├─sda1                8:1    0 74.5G  0 part /
├─sda2                8:2    0    1K  0 part 
├─sda5                8:5    0  9.3G  0 part /var
├─sda6                8:6    0  9.3G  0 part /tmp
└─sda7                8:7    0  6.9G  0 part /boot
sdb                   8:16   0  300G  0 disk 
└─sdb1                8:17   0  300G  0 part 
  └─data_vg-data_lv 254:0    0  300G  0 lvm  /data
  
  
# 这里是单纯划盘，无逻辑扩展能力的裸数据盘。
root@debian:~# lsblk 
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  100G  0 disk 
├─sda1   8:1    0 74.5G  0 part /
├─sda2   8:2    0    1K  0 part 
├─sda5   8:5    0  9.3G  0 part /var
├─sda6   8:6    0  9.3G  0 part /tmp
└─sda7   8:7    0  6.9G  0 part /boot
sdb      8:16   0  200G  0 disk 
└─sdb1   8:17   0  200G  0 part /data

# 无论哪一种，请根据自己的情况使用即可。
```



**第一种：一块磁盘**

> 如果只有一块磁盘，并且您希望所有的 MinIO 实例都使用这块磁盘，那么您可以为每个实例创建一个目录来模拟多个磁盘。这里是如何在 `docker-compose.yaml` 文件中配置单磁盘情况的示例

```shell
services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - /mnt/disk1/minio1/data1:/data1
      - /mnt/disk1/minio1/data2:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - /mnt/disk1/minio2/data1:/data1
      - /mnt/disk1/minio2/data2:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - /mnt/disk1/minio3/data1:/data1
      - /mnt/disk1/minio3/data2:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - /mnt/disk1/minio4/data1:/data1
      - /mnt/disk1/minio4/data2:/data2
```



**第二种：两块磁盘**

> 所有 MinIO 实例都共享这两个磁盘，每个实例使用磁盘的不同目录。

```yaml
services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - /mnt/disk1/part1:/data1
      - /mnt/disk2/part1:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - /mnt/disk1/part2:/data1
      - /mnt/disk2/part2:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - /mnt/disk1/part3:/data1
      - /mnt/disk2/part3:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - /mnt/disk1/part4:/data1
      - /mnt/disk2/part4:/data2
```



**第三种：四块磁盘**

> 每两个 MinIO 实例共享一个磁盘，每个实例使用磁盘的不同目录。

```yaml
services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - /mnt/disk1/part1:/data1
      - /mnt/disk2/part1:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - /mnt/disk1/part2:/data1
      - /mnt/disk2/part2:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - /mnt/disk3/part1:/data1
      - /mnt/disk4/part1:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - /mnt/disk3/part2:/data1
      - /mnt/disk4/part2:/data2
```



**第四种：八块磁盘**

> 每个 MinIO 实例将直接映射到两个磁盘的路径。

```yaml
services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - /mnt/disk1:/data1
      - /mnt/disk2:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - /mnt/disk3:/data1
      - /mnt/disk4:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - /mnt/disk5:/data1
      - /mnt/disk6:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - /mnt/disk7:/data1
      - /mnt/disk8:/data2

```



### 1.3 查看数据



```shell
# 因为我这里只有一块磁盘，所以看到数据的都在同个数据盘里
root@deepin-1:~/minio-cluster# ls -lha /data/minio*/*/*
/data/minio1/data2/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf

/data/minio2/data1/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf

/data/minio2/data2/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf

/data/minio3/data1/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf

/data/minio3/data2/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf

/data/minio4/data1/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf

/data/minio4/data2/demo:
total 12K
drwxr-xr-x 3 root root 4.0K Jan 19 12:04 .
drwxr-xr-x 4 root root 4.0K Jan 19 12:03 ..
drwxr-xr-x 2 root root 4.0K Jan 19 12:04 nginx.conf
```





