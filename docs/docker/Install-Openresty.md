

## Compose file

```yaml
version: '3'
services:
  openresty:
    image: harbor.dockerregistry.com/app/openresty:1.21.4.1-3-alpine
    container_name: openresty
    ports:
      - 8868:8868
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./routerRebuild.lua:/usr/local/openresty/nginx/luapackage/routerRebuild.lua
      - ./nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf
    restart: always
```





## nginx.conf

```shell
worker_processes  auto;
error_log  logs/error.log  info;
events {
    worker_connections  4096;
}
http {
    default_type  application/octet-stream;
    access_log  logs/access.log;
    keepalive_requests 8192;
    keepalive_timeout 180;
    client_body_buffer_size 2000m;
    client_max_body_size 2000m;
    log_format  proxy   '$remote_addr - $remote_user [$time_local] "$request" - $http_host '
    'status:$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
    lua_package_path '/usr/local/openresty/nginx/luapackage/?.lua;;';
    server {
        listen       8868;
        server_name  openrestry.com;
        default_type text/html;
        client_header_buffer_size 1024k;
        large_client_header_buffers 4 1024k;
        fastcgi_buffers             6 512k;
        fastcgi_busy_buffers_size   512k;
        fastcgi_temp_file_write_size        512k;
        fastcgi_intercept_errors    on;
        charset utf-8;
        location = /favicon.ico {
            client_body_buffer_size 100m;
            log_not_found off;
            access_log off;
        }

        location / {
            resolver 1.1.1.1;
            set $target '';
            access_by_lua '
                    -- ngx.say("<p>hello, world</p>")
                    local routerUtil = require ("routerRebuild")
                    ngx.var.target = routerUtil.buildURL()
            ';
            proxy_pass $target;
            proxy_set_header Host  $proxy_host;
            proxy_http_version 1.1;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Connection, User-Agent, Cookie';
            proxy_connect_timeout 3000;
            proxy_set_header Connection keep-alive;
            proxy_send_timeout 3000;
            proxy_read_timeout 6000;
        }
    }
}

```

