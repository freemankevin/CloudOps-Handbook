## 1 change hostname

```bash
# on etcd1
hostnamectl set-hostname etcd1
# on etcd2
hostnamectl set-hostname etcd2
# on etcd3
hostnamectl set-hostname etcd3
```







## 2 no passwd to login

```bash
# On etcd1
cd ~
vim /etc/hosts
# add dns
192.168.171.136 etcd1
192.168.171.137 etcd2
192.168.171.138 etcd3

# ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
sshpass -p '3qjQTv4ytU' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@etcd1"
sshpass -p 'kplDP86PdP' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@etcd2"
sshpass -p 'iWmtmQWWR0' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@etcd3"

```



## 3 add api version

```bash
# On aLL etcd nodes
echo 'export ETCDETC_API=3' | tee -a /etc/profile
source /etc/profile
grep ETCDETC_API /etc/profile

```



## 4 install cfssl

```bash
# On aLL etcd nodes
# install cfssl
# https://github.com/cloudflare/cfssl/releases
\cp -rvf cfssl_1.6.3_linux_amd64     /usr/local/bin/cfssl 
\cp -rvf cfssljson_1.6.3_linux_amd64 /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl
chmod +x /usr/local/bin/cfssljson
cfssl version
cfssljson -version 
```



## 5 install etcd

```bash
# On aLL etcd nodes
# install etcd for three hosts.

tar Cxzvf /tmp etcd-v3.5.8-linux-amd64.tar.gz 
\cp -rvf /tmp/etcd-v3.5.8-linux-amd64/etcd* /usr/local/bin/

etcd --version
etcdctl version
```



## 6 create tls ca

```bash
# On etcd1
# create tls ca
mkdir -p /etc/etcd/ssl
cd /etc/etcd/ssl

cat <<EOF | tee ca-config.json
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "server": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "client": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF


cat <<EOF | tee etcd-ca-csr.json
{
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF




cat <<EOF | tee client-csr.json
{
    "CN": "client",
    "key": {
        "algo": "rsa",
        "size": 2048
    }
}
EOF


# ! notes: Use your own etcd node IP address
cat <<EOF | tee etcd-server-csr.json
{
    "CN": "etcd-server",
    "hosts": [
        "192.168.171.136",
        "192.168.171.137",
        "192.168.171.138",
        "127.0.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "O": "etcd-cluster",
            "OU": "System",
            "ST": "Beijing"
        }
    ]
}
EOF

# ! notes: Use your own etcd node IP address
cat <<EOF | tee etcd-peer-csr.json
{
    "CN": "etcd-peer",
    "hosts": [
        "192.168.171.136",
        "192.168.171.137",
        "192.168.171.138",
        "127.0.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "O": "etcd-cluster",
            "OU": "System",
            "ST": "Beijing"
        }
    ]
}
EOF


## 生成CA证书和私钥
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca - 
## 生成客户端证书
cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem -config=ca-config.json -profile=client client-csr.json      | cfssljson -bare client - 
### 可忽略hosts 未配置警告：
### [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for websites. 
### For more information see the Baseline Requirements for the Issuance and Management of Publicly-Trusted Certificates, v.1.1.6, 
### from the CA/Browser Forum (https://cabforum.org); specifically, section 10.2.3 ("Information Requirements").'

## 生成server，peer证书
cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem -config=ca-config.json -profile=server etcd-server-csr.json | cfssljson -bare etcd-server 
cfssl gencert -ca=etcd-ca.pem -ca-key=etcd-ca-key.pem -config=ca-config.json -profile=peer   etcd-peer-csr.json   | cfssljson -bare etcd-peer


# all files
$ tree /etc/etcd/ssl/
/etc/etcd/ssl/
├── ca-config.json
├── client.csr
├── client-csr.json
├── client-key.pem
├── client.pem
├── etcd-ca.csr
├── etcd-ca-csr.json
├── etcd-ca-key.pem
├── etcd-ca.pem
├── etcd-peer.csr
├── etcd-peer-csr.json
├── etcd-peer-key.pem
├── etcd-peer.pem
├── etcd-server.csr
├── etcd-server-csr.json
├── etcd-server-key.pem
└── etcd-server.pem

0 directories, 17 files

$ tree /etc/etcd/ssl/ | grep pem
├── client-key.pem
├── client.pem
├── etcd-ca-key.pem
├── etcd-ca.pem
├── etcd-peer-key.pem
├── etcd-peer.pem
├── etcd-server-key.pem
└── etcd-server.pem
```



## 7 scp ca files

```bash
# on etcd1 
# scp files

scp -r /etc/etcd  etcd2:/etc/
scp -r /etc/etcd  etcd3:/etc/
```



## 8 create service config

```bash
# create service config
### rhel: /usr/lib/systemd/system/etcd.service 
### debian:  /lib/systemd/system/etcd.service

# etcd and apiserver are in same host. 
# use "Before=kube-apiserver.service" 


## on etcd1 
# ! notes: Use your own etcd node & cluster IP address
$ mkdir -p /data/etcd/ && vim /lib/systemd/system/etcd.service

[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Before=kube-apiserver.service
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/data/etcd/
ExecStart=/usr/local/bin/etcd \
 --name=etcd1 \
 --cert-file=/etc/etcd/ssl/etcd-server.pem \
 --key-file=/etc/etcd/ssl/etcd-server-key.pem \
 --peer-cert-file=/etc/etcd/ssl/etcd-peer.pem \
 --peer-key-file=/etc/etcd/ssl/etcd-peer-key.pem \
 --trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \
 --peer-trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \
 --initial-advertise-peer-urls=https://192.168.171.136:2380 \
 --listen-peer-urls=https://192.168.171.136:2380 \
 --listen-client-urls=https://192.168.171.136:2379 \
 --advertise-client-urls=https://192.168.171.136:2379 \
 --initial-cluster-token=etcd-cluster-0 \
 --initial-cluster=etcd1=https://192.168.171.136:2380,etcd2=https://192.168.171.137:2380,etcd3=https://192.168.171.138:2380 \
 --initial-cluster-state=new \
 --data-dir=/data/etcd \
 --snapshot-count=50000 \
 --auto-compaction-retention=1 \
 --max-request-bytes=10485760 \
 --quota-backend-bytes=8589934592
Restart=always
RestartSec=15
LimitNOFILE=65536
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target



## on etcd2
# ! notes: Use your own etcd node & cluster IP address
$ mkdir -p /data/etcd/ && vim /lib/systemd/system/etcd.service

[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Before=kube-apiserver.service
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/data/etcd/
ExecStart=/usr/local/bin/etcd \
  --name=etcd2 \
  --cert-file=/etc/etcd/ssl/etcd-server.pem \
  --key-file=/etc/etcd/ssl/etcd-server-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd-peer.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-peer-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \
  --initial-advertise-peer-urls=https://192.168.171.137:2380 \
  --listen-peer-urls=https://192.168.171.137:2380 \
  --listen-client-urls=https://192.168.171.137:2379 \
  --advertise-client-urls=https://192.168.171.137:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd1=https://192.168.171.136:2380,etcd2=https://192.168.171.137:2380,etcd3=https://192.168.171.138:2380 \
  --initial-cluster-state=new \
  --data-dir=/data/etcd \
  --snapshot-count=50000 \
  --auto-compaction-retention=1 \
  --max-request-bytes=10485760 \
  --quota-backend-bytes=8589934592
Restart=always
RestartSec=15
LimitNOFILE=65536
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target






## on  etcd3 
# ! notes: Use your own etcd node & cluster IP address
$ mkdir -p /data/etcd/ && vim /lib/systemd/system/etcd.service

[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Before=kube-apiserver.service
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/data/etcd/
ExecStart=/usr/local/bin/etcd \
  --name=etcd3 \
  --cert-file=/etc/etcd/ssl/etcd-server.pem \
  --key-file=/etc/etcd/ssl/etcd-server-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd-peer.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-peer-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/etcd-ca.pem \
  --initial-advertise-peer-urls=https://192.168.171.138:2380 \
  --listen-peer-urls=https://192.168.171.138:2380 \
  --listen-client-urls=https://192.168.171.138:2379 \
  --advertise-client-urls=https://192.168.171.138:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd1=https://192.168.171.136:2380,etcd2=https://192.168.171.137:2380,etcd3=https://192.168.171.138:2380 \
  --initial-cluster-state=new \
  --data-dir=/data/etcd \
  --snapshot-count=50000 \
  --auto-compaction-retention=1 \
  --max-request-bytes=10485760 \
  --quota-backend-bytes=8589934592
Restart=always
RestartSec=15
LimitNOFILE=65536
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target

```



## 9 start service

```bash
# on all etcd nodes

# start service 
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd 
systemctl status etcd -l 
```



## 10 view logs

```bash  
# on all etcd nodes

# other terminal query logs
journalctl -xeu etcd -f

```



## 11 add alias

```bash
# on etcd1 
# add alias
vim ~/.bashrc
# add alias 
# ! notes: Use your own etcd cluster IP address
alias etcdctl='etcdctl \
        --endpoints=192.168.171.136:2379,192.168.171.137:2379,192.168.171.138:2379 \
        --cacert=/etc/etcd/ssl/etcd-ca.pem \
        --cert=/etc/etcd/ssl/etcd-server.pem   \
        --key=/etc/etcd/ssl/etcd-server-key.pem '

source ~/.bashrc


# scp to other nodes.
scp -r ~/.bashrc etcd2:~/
scp -r ~/.bashrc etcd3:~/

ssh etcd2 "source ~/.bashrc"
ssh etcd3 "source ~/.bashrc"
```



## 12 endpoint status & health

```bash
# on all etcd node.

# 查看选主
$ etcdctl endpoint status -w table
+----------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|       ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.171.136:2379 | 63a24950535b517e |   3.5.8 |   20 kB |      true |      false |         2 |          9 |                  9 |        |
| 192.168.171.137:2379 | 6419b1a69bbf5747 |   3.5.8 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
| 192.168.171.138:2379 | b14454725168d230 |   3.5.8 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
+----------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

# 健康检查
$ etcdctl endpoint health -w table
+----------------------+--------+-------------+-------+
|       ENDPOINT       | HEALTH |    TOOK     | ERROR |
+----------------------+--------+-------------+-------+
| 192.168.171.136:2379 |   true |  9.455597ms |       |
| 192.168.171.137:2379 |   true |  9.162171ms |       |
| 192.168.171.138:2379 |   true | 12.529629ms |       |
+----------------------+--------+-------------+-------+
```





## 13 input test

```bash
# on any etcd nodes.

# 写入测试
$ etcdctl put lxs "test"
OK

# 查看数据
$ etcdctl get lxs
lxs
test

# 删除测试
$ etcdctl del lxs
1
$ etcdctl get lxs


# 查看key
$ etcdctl  --prefix --keys-only=true get /
```



## 13 Add Backup Rules

```bash
# on etcd3 
mkdir -p /data/etcd-backups/ && cd /data/etcd-backups/


# add script 
# change here ip in yourself
tee etcd-backups.sh <<-EOF
#!/bin/bash

# ETCD backup directory
BACKUP_DIR=/data/etcd-backups

# Current date and time
DATE=$(date +%Y%m%d%H%M%S)

# Log file
LOG_FILE=${BACKUP_DIR}/backup-${DATE}.log

# ETCDCTL command
ETCDCTL_CMD="/usr/local/bin/etcdctl \
    --endpoints=192.168.1.120:2379 \
    --cacert=/etc/etcd/ssl/etcd-ca.pem \
    --cert=/etc/etcd/ssl/etcd-server.pem \
    --key=/etc/etcd/ssl/etcd-server-key.pem"

# ETCDCTL API version
export ETCDCTL_API=3

# ETCD backup command
echo "Starting backup at ${DATE}" > ${LOG_FILE}
${ETCDCTL_CMD} snapshot save ${BACKUP_DIR}/backup-${DATE}.db >> ${LOG_FILE} 2>&1

# Check if the backup was successful
if [ $? -eq 0 ]; then
    echo "Backup successful" >> ${LOG_FILE}
else
    echo "Backup failed" >> ${LOG_FILE}
fi

# Remove backups older than 7 days
find ${BACKUP_DIR} -name "backup-*.db" -mtime +7 -exec rm {} \; >> ${LOG_FILE} 2>&1

# Remove logs older than 7 days
find ${BACKUP_DIR} -name "backup-*.log" -mtime +7 -exec rm {} \; >> ${LOG_FILE} 2>&1
EOF


# chmod
chmod +x etcd-backups.sh

# test 
./etcd-backups.sh


# add cron
root@etcd1:/data/etcd-backups# crontab -e
crontab: installing new crontab

root@etcd1:/data/etcd-backups# crontab -l 
0 2 * * * /data/etcd-backups/etcd-backups.sh
```

