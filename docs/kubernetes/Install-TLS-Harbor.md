## 1 Change DNS

```bash
# in windows & clash proxy & add dns
hosts:
  'harbor.dockerregistry.com': '192.168.171.50' # here

# in windows & hosts
192.168.171.50 harbor.dockerregistry.com


# harbor node.
hostnamectl set-hostname harbor.dockerregistry.com

echo -e "\n192.168.171.50 harbor.dockerregistry.com" >> /etc/hosts

```



## 2 Create lvm

```bash
# 安装包
sudo apt-get update
sudo apt-get install lvm2

# 创建物理卷
# 使用pvcreate命令在sdb磁盘上创建一个物理卷：
pvcreate /dev/sdb1

# 创建卷组
# 使用vgcreate命令在刚刚创建的物理卷上创建一个卷组。在这个例子中，我们将卷组命名为data_vg：
vgcreate data_vg /dev/sdb1

# 创建逻辑卷
# 现在，你可以在卷组data_vg上创建逻辑卷。在这个例子中，我们将逻辑卷命名为data_lv，并分配所有可用的空间：
lvcreate -l +100%FREE -n data_lv data_vg

# 格式化逻辑卷
# 在逻辑卷上创建一个文件系统。这里我们使用ext4文件系统，你也可以根据需要选择其它的文件系统：
mkfs.ext4 /dev/data_vg/data_lv

# 清理之前的旧配置
sed -i '/\/data/d' /etc/fstab

# 重启服务器
sync;reboot

# 挂载逻辑卷
# 最后，你可以将新的逻辑卷挂载到/data目录：
mount /dev/data_vg/data_lv /data

# 更新/etc/fstab文件
# 为了在系统重启后保持这个挂载，你需要将新配置添加到/etc/fstab文件中：
echo "/dev/data_vg/data_lv /data ext4 defaults 0 0" >> /etc/fstab

# 刷新挂载
mount -a
```



## 3 Install Docker

```bash
# 更新现有的包列表
sudo apt-get update
# 安装所需的包以让apt支持HTTPS
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release
# 添加Docker的官方GPG密钥
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# 使用下面的命令设置稳定版的仓库
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# 再次更新apt包索引
sudo apt-get update 
# 安装最新版本的Docker Engine和containerd
sudo apt-get install docker-ce docker-ce-cli containerd.io

# 添加配置
cat > /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": ["https://ustc-edu-cn.mirror.aliyuncs.com"],
  "debug": false,
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

# 重启服务
systemctl restart docker 


# 安装compose
cp docker-compose-linux-x86_64 /usr/bin/docker-compose
chmod +x /usr/bin/docker-compose
docker-compose version 

```



## 4 Unpack Harbor

```bash
# 释放安装包
tar xzvf harbor-offline-installer-v2.8.4.tgz -C /data

# 添加配置
\cp -rvf /data/harbor/harbor.yml.tmpl /data/harbor/harbor.yml

# 修改配置
vim /data/harbor/harbor.yml
# 主要修改这6 行配置
hostname: harbor.dockerregistry.com
  certificate: /data/harbor/ssl/harbor.dockerregistry.com.crt
  private_key: /data/harbor/ssl/harbor.dockerregistry.com.key
harbor_admin_password: Harbor123#@!
data_volume: /data/harbor/harbor-data
    location: /data/harbor/harbor-logs
```



## 5 Create ca files

```bash
# 证书制作
mkdir -p  /data/harbor/ssl && cd /data/harbor/ssl
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.dockerregistry.com" \
 -key ca.key \
 -out ca.crt

openssl genrsa -out harbor.dockerregistry.com.key 4096
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.dockerregistry.com" \
    -key harbor.dockerregistry.com.key \
    -out harbor.dockerregistry.com.csr

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.dockerregistry.com
EOF



openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.dockerregistry.com.csr \
    -out harbor.dockerregistry.com.crt

openssl x509 -inform PEM -in harbor.dockerregistry.com.crt -out harbor.dockerregistry.com.cert

# 证书拷贝
mkdir -p /etc/docker/certs.d/harbor.dockerregistry.com/
cp ca.crt /etc/docker/certs.d/harbor.dockerregistry.com/
cp harbor.dockerregistry.com.cert  /etc/docker/certs.d/harbor.dockerregistry.com/
cp harbor.dockerregistry.com.key   /etc/docker/certs.d/harbor.dockerregistry.com/


# 证书发送到客户端(若有客户端需要走HTTPS 访问Harbor的话，则需要发送)
scp -r /etc/docker/certs.d 192.168.171.130:/etc/docker
```



## 6 Install Harbor

```bash
cd /data/harbor/

# 导入离线镜像包
docker load -i harbor.v2.8.4.tar.gz 
# 环境检查
./prepare --with-trivy
# 安装
./install.sh --with-trivy 

# 停止服务（若需要）
docker-compose down


# 登陆测试
docker login -u admin -p Harbor123#@! harbor.dockerregistry.com
```



## 7 Update Client

```bash
# Docker 配置
# 通过HTTP 登陆Harbor 配置
cat > /etc/docker/daemon.json << 'EOF'
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

# 通过HTTPS 登陆Harbor 配置
# 需要提前在服务器上面放Harbor 的证书才行，与前面Harbor 证书发送到客户端同理
cat > /etc/docker/daemon.json << 'EOF'
{
  "registry-mirrors": ["https://ustc-edu-cn.mirror.aliyuncs.com"],
  "debug": false,
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

# 重启docker
systemctl restart docker

# 登录认证
$ docker login -u admin -p Harbor123#@! harbor.dockerregistry.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

