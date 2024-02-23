

## 部署

```shell
#### 服务端

# 1.安装nfs（服务端与客户端等同）（所有节点）
yum install -y nfs-utils

# 2.执行命令 vi /etc/exports，创建 exports 文件
# 文件内容如下：
echo "/data/nfs-share *(insecure,rw,sync,no_root_squash)" > /etc/exports
# 查看配置
cat /etc/exports 

# 3.执行以下命令，启动 nfs 服务
# 创建共享目录
mkdir -p /data/nfs-share

# 服务自启
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server

# nfs重新挂载
exportfs -r

# 检查配置是否生效
exportfs


# 效果如下所示即为OK
/data/nfs-share
								 	<world>


### 客户端
showmount -e  192.168.1.1 
mkdir  /data/share-file   # 这个目录主要是去 共享服务端分享的那个目录，同步那个文件
mount -t nfs 192.168.1.1:/data/nfs-share /data/share-file   #这个是挂载
```



## 挂载时遇到不支持

>`mount.nfs: Protocol not supported`
```shell
systemctl restart rpcbind.service # 重启服务
yum reinstall  nfs-utils  # 重装客户端
```
