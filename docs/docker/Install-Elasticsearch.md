## Compose file

```yaml
version: '3'
services:
  elasticsearch:
    image: harbor.dockerregistry.com/app-arm/elasticsearch:6.5.4
    container_name: elasticsearch
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./elastic/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /data/middleware/es/plugins:/usr/share/elasticsearch/plugins
      - /data/middleware/es/data:/usr/share/elasticsearch/data
    ports:
        - "9200:9200"
        - "9300:9300"
    environment:
        - discovery.type=single-node
    restart: always
```





## elasticsearch.yml

```yaml
bootstrap.memory_lock: true
cluster.name: "es-server"
node.name: single-node
node.master: true
node.data: true
network.host: 0.0.0.0
http.port: 9200
path.logs: /usr/share/elasticsearch/logs
http.cors.enabled: true
http.cors.allow-origin: "*"
xpack.security.audit.enabled: false
xpack.ml.enabled: false
```

