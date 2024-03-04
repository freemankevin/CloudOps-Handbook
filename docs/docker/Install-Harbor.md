

## 1 Harbor For TLS

[Harbor官方文档](https://goharbor.io/docs/2.10.0/install-config/configure-https/)

### 1.1 证书配置
```shell
$ grep myharbor ~/harbor/harbor.yml 
hostname: myharbor.mydomain.com
  certificate: /data/harbor/ssl/myharbor.mydomain.com.crt
  private_key: /data/harbor/ssl/myharbor.mydomain.com.key
```

### 1.2 证书制作
```shell
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=myharbor.mydomain.com" \
 -key ca.key \
 -out ca.crt

openssl genrsa -out myharbor.mydomain.com.key 4096
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=myharbor.mydomain.com" \
    -key myharbor.mydomain.com.key \
    -out myharbor.mydomain.com.csr

cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=myharbor.mydomain.com
EOF



openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in myharbor.mydomain.com.csr \
    -out myharbor.mydomain.com.crt

openssl x509 -inform PEM -in myharbor.mydomain.com.crt -out myharbor.mydomain.com.cert
```

### 1.3 证书拷贝
```shell
mkdir -p /etc/docker/certs.d/myharbor.mydomain.com/
cp ca.crt /etc/docker/certs.d/myharbor.mydomain.com/
cp myharbor.mydomain.com.cert  /etc/docker/certs.d/myharbor.mydomain.com/
cp myharbor.mydomain.com.key   /etc/docker/certs.d/myharbor.mydomain.com/
```

### 1.4 更新系统证书
```shell
\cp -rvf myharbor.mydomain.com.crt /usr/local/share/ca-certificates/myharbor.mydomain.com.crt 
update-ca-certificates
```

### 1.5 证书发送到客户端
```shell
scp -r /etc/docker/certs.d 192.168.171.130:/etc/docker
scp -r /etc/docker/certs.d 192.168.171.132:/etc/docker
scp -r /etc/docker/certs.d 192.168.171.133:/etc/docker
scp -r myharbor.mydomain.com.crt 192.168.171.133:/usr/local/share/ca-certificates/
```

### 1.6 Docker 配置
```shell
cat > /etc/docker/daemon.json <<-EOF
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
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

### 1.7 重启docker
```shell
systemctl restart docker
update-ca-certificates
```

### 1.8 登录认证
```shell
$ docker login -u admin -p Harbor12345 myharbor.mydomain.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```