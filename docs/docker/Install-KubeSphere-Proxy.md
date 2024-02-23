## Proxy

```bash
# https://ask.kubesphere.io/forum/d/1075-kubersphere/4


# Add deploy files.
root@harbor:~# mkdir -p /data/nginx/ && cd /data/nginx/
# upload files 

root@harbor:/data/nginx# tree ks-console
ks-console
├── docker-compose.yaml
└── nginx
    ├── conf.d
    │   └── ks-console.conf
    ├── nginx.conf
    └── ssl
        ├── kubesphere.k8scluster.com.crt
        └── kubesphere.k8scluster.com.key

3 directories, 5 files


# Start svc
root@harbor:/data/nginx# cd ks-console/
                                                                                                                                                                                               0.1s 
root@harbor:/data/nginx/ks-console# docker-compose up -d 
[+] Running 2/2
 ✔ Network ks-console_default  Created                                                                                                                                                                                                0.0s 
 ✔ Container ks-console-proxy  Started 


# Stop svc (if need.)
root@harbor:/data/nginx/ks-console# docker-compose down 
[+] Running 2/2
 ✔ Container ks-console-proxy  Removed                                                                                                                                                                                                0.2s 
 ✔ Network ks-console_default  Removed 


# Status svc
root@harbor:/data/nginx/ks-console# docker-compose ps -a 
NAME               IMAGE                                                   COMMAND                  SERVICE   CREATED              STATUS                        PORTS
ks-console-proxy   harbor.dockerregistry.com/library/nginx:1.21.6-alpine   "/docker-entrypoint.…"   web       About a minute ago   Up About a minute (healthy)   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 80/tcp, 0.0.0.0:8443->8443/tcp, :::8443->8443/tcp
root@harbor:/data/nginx/ks-console# 

```



## logs rotate

```bash
tee /etc/logrotate.d/nginx <<-eof
/data/nginx/logs/access.log /data/nginx/logs/error.log {
    daily
    rotate 7
    missingok
    dateext
    compress
    notifempty
    create 0644 root root
    sharedscripts
    postrotate
        /usr/bin/docker-compose -f /data/nginx/ks-console/docker-compose.yaml exec -T web nginx -s reload
    endscript
}
eof

root@harbor:~/ks-console# cat /etc/logrotate.d/nginx 
/data/nginx/logs/access.log /data/nginx/logs/error.log {
    daily
    rotate 7
    missingok
    dateext
    compress
    notifempty
    create 0644 root root
    sharedscripts
    postrotate
        /usr/bin/docker-compose -f /data/nginx/ks-console/docker-compose.yaml exec -T web nginx -s reload
    endscript
}


# test logrotate
root@harbor:/data/nginx/ks-console# /usr/sbin/logrotate -d /etc/logrotate.d/nginx
WARNING: logrotate in debug mode does nothing except printing debug messages!  Consider using verbose mode (-v) instead if this is not what you want.

reading config file /etc/logrotate.d/nginx
Reading state from file: /var/lib/logrotate/status
Allocating hash table for state file, size 64 entries
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state
Creating new state

Handling 1 logs

rotating pattern: /data/nginx/logs/access.log /data/nginx/logs/error.log  after 1 days (7 rotations)
empty log files are not rotated, old logs are removed
considering log /data/nginx/logs/access.log
  Now: 2023-12-06 10:06
  Last rotated at 2023-12-06 10:00
  log does not need rotating (log has already been rotated)
considering log /data/nginx/logs/error.log
  Now: 2023-12-06 10:06
  Last rotated at 2023-12-06 10:00
  log does not need rotating (log has already been rotated)
not running postrotate script, since no logs were rotated
```



## cron

```bash
root@harbor:~/ks-console# crontab -e
crontab: installing new crontab

root@harbor:~/ks-console# crontab -l
0 2 * * * /usr/sbin/logrotate /etc/logrotate.d/nginx
```





## ks-console.conf

```shell
upstream backend {
    server 10.0.0.1:30880 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:30880 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:30880 max_fails=3 fail_timeout=30s;
}

# HTTP server
server {
    listen 80;

    server_name kubesphere.k8scluster.com;

    location / {
        return 301 https://$host:443$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;

    server_name kubesphere.k8scluster.com;

    #include ssl-conf/ssl-full.loadttl.com.conf;

    access_log /var/log/nginx/ks-console-access.log main;
    error_log  /var/log/nginx/ks-console-error.log;

    index index.html index.htm;

    ssl_certificate     /etc/nginx/ssl/ks-console/kubesphere.k8scluster.com.crt;
    ssl_certificate_key /etc/nginx/ssl/ks-console/kubesphere.k8scluster.com.key;

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

    index index.html index.htm;
    if ($ssl_protocol = "") { return 301 https://$host$request_uri; }

    # Define global proxy settings
    proxy_http_version 1.1;
    proxy_set_header    Host $host:$server_port;
    proxy_set_header    X-Real-IP $remote_addr;
    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_connect_timeout  3600s;
    proxy_read_timeout  3600s;
    proxy_send_timeout  3600s;
    send_timeout  3600s;

    location / {
        proxy_pass http://backend;
        proxy_redirect off;
    }

    location /api/ {
        proxy_redirect off;
        proxy_pass http://backend;
        proxy_set_header    Upgrade $http_upgrade;
        proxy_set_header    X-Forwarded-Proto $scheme;
        proxy_set_header    Connection "upgrade";
    }

    location ~ /apis/monitoring.coreos.com/|/api/v1/|/apis/storage.k8s.io|/apis/apps/v1/namespaces/|/kapis/resources.kubesphere.io/v1alpha2/namespaces|/kapis/resources.kubesphere.io/|/apis/devops.kubesphere.io/|/apis/apps/v1/|/apis/|/api/v1/watch/namespaces|/kapis/terminal.kubesphere.io/ {
        proxy_pass http://backend;
        proxy_redirect off;
    }
}
```

