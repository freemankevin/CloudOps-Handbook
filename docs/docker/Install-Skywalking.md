## Compose file

```yaml
version: '3.8'
services:
  elasticsearch:
    image: harbor.dockerregistry.com/app/elasticsearch:7.17.1
    container_name: elasticsearch4
    ports:
      - "9200:9200"
    deploy:
      resources:
        limits:
          cpus: '0.80'
          memory: 1000M
    healthcheck:
      test: [ "CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1" ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /data/es:/usr/share/elasticsearch/data
    restart: always

  oap:
    image: harbor.dockerregistry.com/app/skywalking-oap-server:8.9.0 
    container_name: oap
    deploy:
      resources:
        limits:
          cpus: '0.80'
          memory: 2000M
    depends_on:
      elasticsearch:
       condition: service_healthy
    links:
      - elasticsearch
    ports:
      - "11800:11800"
      - "12800:12800"
    healthcheck:
      test: [ "CMD-SHELL", "/skywalking/bin/swctl ch" ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    environment:
      SW_STORAGE: elasticsearch
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      SW_HEALTH_CHECKER: default
      SW_TELEMETRY: prometheus
      JAVA_OPTS: "-Xms2048m -Xmx2048m"
    restart: always

  ui:
    image: harbor.dockerregistry.com/app/skywalking-ui:8.9.0
    container_name: ui
    deploy:
      resources:
        limits:
          cpus: '0.90'
          memory: 2000M
    depends_on:
      oap:
        condition: service_healthy
    links:
      - oap
    ports:
      - "8080:8080"
    environment:
      SW_OAP_ADDRESS: http://oap:12800
    restart: always

```