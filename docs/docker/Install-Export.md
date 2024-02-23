## Node_Exporter
> 物理方式，`debian` 系统

```shell
\cp -rvf  node_exporter-1.7.0.linux-amd64/node_exporter /usr/bin/node_exporter 
chmod +x /usr/bin/node_exporter

sudo tee /lib/systemd/system/node_exporter.service <<-'EOF'
[Unit]
Description=node_exporter Monitoring System
After=network.target

[Service]
ExecStart=/usr/bin/node_exporter --web.listen-address=:19100

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

> Compose 方式

```yaml
version: '3.8'
services:
  linux_exporter:
    image: harbor.dockerregistry.com/app/linux_exporter:latest
    container_name: linux_exporter
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    pid: host
    volumes:
      - '/:/host:ro,rslave'
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 100M
    restart: always
    privileged: true
```

## Docker_Exporter
```yaml
version: '3.8'
services:
  docker_exporter:
    image: harbor.dockerregistry.com/app/docker_exporter:latest
    container_name: docker_exporter
    ports:
        - "19104:8080"
    volumes:
      - '/:/rootfs:ro'
      - '/sys:/sys:ro'
      - '/var/run:/var/run:ro'
      - '/dev/disk/:/dev/disk:ro'
      - '/var/lib/docker/:/var/lib/docker:ro'
    devices:
      - '/dev/kmsg:/dev/kmsg'
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 100M
    restart: always
    privileged: true
```


## Postgres_Exporter
```yaml
version: '3.8'
services:
  postgres_exporter:
    image: harbor.dockerregistry.com/app/postgres_exporter:latest
    container_name: postgres_exporter
    ports:
      - "9187:9187"
    environment:
      DATA_SOURCE_NAME: postgresql://postgres:postgres@10.1.6.239:5432/postgres?sslmode=disable
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 100M
    restart: always
```


## Redis-Exporter
```yaml
version: '3.8'
services:
  redis-exporter:
    image: harbor.dockerregistry.com/app/quay.io/oliver006/redis_exporter
    container_name: redis-exporter
    ports:
      - "9121:9121"
    environment:
      redis.password: sso123456
    restart: always
    privileged: true
```

## Oracledb-Exporter
```yaml
version: '3.8'
services:
  oracledb-exporter:
    image: harbor.dockerregistry.com/app/iamseth/oracledb_exporter
    container_name: oracledb-exporter
    ports:
      - "9161:9161"
    environment:
      DATA_SOURCE_NAME: system/123456@10.0.1.186:1521/ORCL
    restart: always
    privileged: true
```

## Gpu-Exporter
```yaml
version: '3.8'
services:
  gpu-exporter:
    image: harbor.dockerregistry.com/app/utkuozdemir/nvidia_gpu_exporter:0.3.0
    container_name: gpu-exporter
    ports:
      - "9835:9835"
    volumes:
      - '/usr/lib/x86_64-linux-gnu/libnvidia-ml.so:/usr/lib/x86_64-linux-gnu/libnvidia-ml.so'
      - '/usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1:/usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1'
      - '/usr/bin/nvidia-smi:/usr/bin/nvidia-smi'
    devices:
      - '/dev/nvidiactl:/dev/nvidiactl'
      - '/dev/nvidia0:/dev/nvidia0'
    restart: always
    privileged: true
```