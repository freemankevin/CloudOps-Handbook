## 1 安装 `Keepalived`

```bash

## 主服务器配置
apt-get update
apt-get install psmisc
apt install -y keepalived

root@debian:~# cat /etc/keepalived/check_apiserver.sh 
#!/bin/bash

API_URL="https://192.168.1.110:6443/healthz"

if curl -k -s $API_URL | grep -q "ok"
then
    exit 0
else
    exit 1
fi


cat <<EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy" 
    interval 2 
    weight 2 
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 51
    priority 101
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.110
    }
    unicast_src_ip 192.168.1.111  # This haproxy node
    unicast_peer {
        192.168.1.112             # Other haproxy nodes
    }
    track_script {
        check_apiserver
        check_haproxy
    }
}
EOF

keepalived -t -f /etc/keepalived/keepalived.conf
systemctl start keepalived
systemctl enable keepalived
systemctl status keepalived


## 备服务器配置
apt-get update
apt-get install psmisc
apt install -y keepalived

root@debian:~# cat /etc/keepalived/check_apiserver.sh 
#!/bin/bash

API_URL="https://192.168.1.110:6443/healthz"

if curl -k -s $API_URL | grep -q "ok"
then
    exit 0
else
    exit 1
fi

cat <<EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy" 
    interval 2 
    weight 2 
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.110
    }
    unicast_src_ip 192.168.1.112  # This haproxy node
    unicast_peer {
        192.168.1.111             # Other haproxy nodes
    }
    track_script {
        check_apiserver
        check_haproxy
    }
}
EOF

keepalived -t -f /etc/keepalived/keepalived.conf

systemctl start keepalived
systemctl enable keepalived
systemctl status keepalived
```



## 2 安装 `Haproxy`

```bash
## deploy haproxy in both vm.
apt install -y haproxy
cat <<EOF | tee /etc/haproxy/haproxy.cfg  
# /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:6443
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server master1 192.168.171.136:6443 check
        server master2 192.168.171.137:6443 check
        server master3 192.168.171.138:6443 check

EOF
systemctl start  haproxy
systemctl enable haproxy
systemctl status haproxy

```





