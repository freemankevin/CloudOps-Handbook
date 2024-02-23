

## Compose file

```yaml
version: '3'
services:
  minio:
    image: harbor.dockerregistry.com/app/minio:latest
    container_name: minio
    ports:
      - 9000:9000
      - 9091:9091
    command: server /data --console-address ":9091"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: admin@123
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /data/middleware/minio/data:/data
```



## Physical deployment

### 1 Install Minio

```bash
# https://min.io/docs/minio/linux/index.html

# wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20231120224007.0.0_amd64.deb -O minio.deb
# upload files
root@harbor:~# ls -lh minio_20231120224007.0.0_amd64.deb 
-rw-r--r-- 1 root root 35M Dec  6 10:33 minio_20231120224007.0.0_amd64.deb

root@harbor:~# dpkg -i minio_20231120224007.0.0_amd64.deb 
Selecting previously unselected package minio.
(Reading database ... 40297 files and directories currently installed.)
Preparing to unpack minio_20231120224007.0.0_amd64.deb ...
Unpacking minio (20231120224007.0.0) ...
Setting up minio (20231120224007.0.0) ...
```





### 2 Install Client (Optional)

```bash
# wget https://dl.min.io/client/mc/release/linux-amd64/mc
# upload file.
root@harbor:~# ls mc 
mc



root@harbor:~# chmod +x mc
root@harbor:~# mv mc /usr/local/bin/mc
root@harbor:~# mc -v 
mc version RELEASE.2023-12-02T02-03-28Z (commit-id=f5f7147b9ec4cf78eb67f1cdc91b63d191852e6a)
Runtime: go1.21.4 linux/amd64
Copyright (c) 2015-2023 MinIO, Inc.
License GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
```



### 3 Create ca

```bash
# https://min.io/docs/minio/linux/operations/install-deploy-manage/upgrade-minio-deployment.html#minio-upgrade-systemctl
# mkdir /data/minio
# minio server /data/minio --console-address :9090

# rootCA.key：CA（证书颁发机构）的私钥
# rootCA.crt：自签名的根证书
# server.key：服务器的私钥
# server.csr：服务器证书签名请求
# server.crt：服务器证书

mkdir -p /etc/minio/ssl/certs && cd /etc/minio/ssl/certs


# change here ip addr to yourself
tee openssl-ca.cnf <<-eof
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
default_days = 3650
default_md = sha256
prompt = no

[req_distinguished_name]
CN = My Root CA

[v3_ca]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical, CA:true
eof

tee openssl-server.cnf <<-eof
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no
default_days = 3650
default_md = sha256

[req_distinguished_name]
CN = minio.objectstorage.com

[v3_req]
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = minio.objectstorage.com
IP.1 = 192.0.2.1
eof

# 制作证书
openssl genrsa -out rootCA.key 2048 # 创建CA的私钥
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.crt -config openssl-ca.cnf # 创建并自签名根证书(CA)

openssl genrsa -out server.key 2048 # 创建服务器的私钥
openssl req -new -key server.key -out server.csr -config openssl-server.cnf # 创建服务器证书签名请求(CSR)
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out server.crt -days 3650 -sha256 -extensions v3_req -extfile openssl-server.cnf


mv server.key private.key
mv server.crt public.crt
```



### 4 Create config

```bash
mkdir -p /data/minio
tee /etc/default/minio <<-eof
# templates/minio.j2
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=admin@123
MINIO_VOLUMES="/data/minio"
#MINIO_SERVER_URL="http://minio.objectstorage.com:9000"
#MINIO_OPTS="--address :9000 --console-address :9001"
MINIO_SERVER_URL="https://minio.objectstorage.com:9000"
MINIO_OPTS="--address :9000 --certs-dir /etc/minio/ssl/certs --console-address :9001"
eof

useradd -r minio-user -s /sbin/nologin
chown -R minio-user:minio-user /data/minio
chown -R minio-user.minio-user /etc/minio 
```



### 5 Start service

```bash
systemctl start minio
systemctl enable minio
```



### 6 Install proxy

```bash
# https://github.com/minio/minio/blob/master/docs/orchestration/docker-compose/nginx.conf

# openssl req -x509 -newkey rsa:2048 -sha256 -nodes \
#     -keyout minio.objectstorage.com.key \
#     -days 3650 \
#     -subj "/C=CN/ST=Guangdong/L=Guangzhou/O=My Organization/OU=My Unit/CN=minio.objectstorage.com" \
#     -out minio.objectstorage.com.crt \
#     -extensions SAN \
#     -config <(echo "[req]"; 
#               echo distinguished_name=req; 
#               echo "[SAN]"; 
#               echo subjectAltName=DNS:minio.objectstorage.com)


# upload files.
root@harbor:/data/nginx/web# tree .
.
├── docker-compose.yaml
└── nginx
    ├── conf.d
    │   └── minio.conf
    ├── nginx.conf
    └── ssl
        └── minio
            ├── minio.objectstorage.com.crt
            └── minio.objectstorage.com.key

5 directories, 8 files


# svc up
root@harbor:/data/nginx/web# docker-compose up -d 
[+] Running 1/0
 ✔ Container proxy  Running   


# svc status
root@harbor:/data/nginx/web# docker-compose ps -a 
NAME      IMAGE                                                   COMMAND                  SERVICE   CREATED         STATUS                   PORTS
proxy     harbor.dockerregistry.com/library/nginx:1.21.6-alpine   "/docker-entrypoint.…"   web       4 minutes ago   Up 4 minutes (healthy)   0.0.0.0:9090->9090/tcp, :::9090->9090/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp, 0.0.0.0:8080->80/tcp, :::8080->80/tcp, 0.0.0.0:8443->443/tcp, :::8443->443/tcp



# logs dir
root@harbor:/data/nginx/web# ls -lh /data/nginx/logs/
total 20K
-rw-r--r-- 1 root root    0 Dec  6 15:49 access.log
-rw-r--r-- 1 root root    0 Dec  6 15:49 error.log
-rw-r--r-- 1 root root 9.1K Dec  6 15:53 ks-console-access.log
-rw-r--r-- 1 root root    0 Dec  6 15:49 ks-console-error.log
-rw-r--r-- 1 root root 5.7K Dec  6 15:53 minio-access.log
-rw-r--r-- 1 root root    0 Dec  6 15:49 minio-error.logs
```



### 7 Compose file

```yaml
version: '3.8'
services:
  web:
    image: harbor.dockerregistry.com/library/nginx:1.21.6-alpine
    extra_hosts:
      - "minio.objectstorage.com:192.168.1.120"
    container_name: proxy
    networks:
      - web
    ports:
      - "8080:80"
      - "9090:9090"
      - "8443:443"
      - "9443:9443"
    volumes:
      - '/etc/localtime:/etc/localtime:ro'
      - './nginx/ssl:/etc/nginx/ssl'
      - './nginx/nginx.conf:/etc/nginx/nginx.conf'
      - './nginx/conf.d/ks-console.conf:/etc/nginx/conf.d/default.conf'
      - './nginx/conf.d/minio.conf:/etc/nginx/conf.d/minio.conf'
      - '/data/nginx/logs:/var/log/nginx'
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl --silent --fail --insecure https://localhost:8443 || curl --silent --fail --insecure https://localhost:9443 || exit 1"]
      interval: 1m30s
      timeout: 10s
      retries: 3
      start_period: 3s

networks:
  web:
    external: false
```



### 8 Nginx.conf

```shell
user  nginx;
worker_processes  auto;

#error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```



### 9 Minio.conf

```shell
upstream minio {
    server minio.objectstorage.com:9000;
}

upstream console {
    ip_hash;
    server minio.objectstorage.com:9001;
}

server {
    listen       9090 ssl;
    listen  [::]:9090 ssl;
    server_name  minio.objectstorage.com;

    access_log /var/log/nginx/minio-access.log main;
    error_log  /var/log/nginx/minio-error.log;

    ssl_certificate     /etc/nginx/ssl/minio/minio.objectstorage.com.crt;
    ssl_certificate_key /etc/nginx/ssl/minio/minio.objectstorage.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    #ssl_stapling on;
    #ssl_stapling_verify on;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    # To allow special characters in headers
    ignore_invalid_headers off;
    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 0;
    # To disable buffering
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 300;
        # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;

        proxy_pass https://minio;
    }
}

server {
    listen       9443 ssl;
    listen  [::]:9443 ssl;
    server_name  minio.objectstorage.com;

    access_log /var/log/nginx/minio-access.log main;
    error_log  /var/log/nginx/minio-error.log;

    ssl_certificate     /etc/nginx/ssl/minio/minio.objectstorage.com.crt;
    ssl_certificate_key /etc/nginx/ssl/minio/minio.objectstorage.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    #ssl_stapling on;
    #ssl_stapling_verify on;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    # To allow special characters in headers
    ignore_invalid_headers off;
    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 0;
    # To disable buffering
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-NginX-Proxy true;

        # This is necessary to pass the correct IP to be hashed
        real_ip_header X-Real-IP;

        proxy_connect_timeout 300;
        
        # To support websocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        chunked_transfer_encoding off;

        proxy_pass https://console;
    }
}
```





