## Compose file

```yaml
version: '3'
services:
  elasticsearch:
    image: harbor.dockerregistry.com/app/elasticsearch:7.17.1
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - discovery.seed_hosts=elasticsearch2,elasticsearch3
      - cluster.initial_master_nodes=elasticsearch,elasticsearch2,elasticsearch3
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /data/es/data01:/usr/share/elasticsearch/data
    ports:
      - 19200:9200
    networks:
      - esnet
    restart: always
  elasticsearch2:
    image: harbor.dockerregistry.com/app/elasticsearch:7.17.1
    container_name: elasticsearch2
    environment:
      - node.name=elasticsearch2
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - discovery.seed_hosts=elasticsearch,elasticsearch3
      - cluster.initial_master_nodes=elasticsearch,elasticsearch2,elasticsearch3
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /data/es/data02:/usr/share/elasticsearch/data
    networks:
      - esnet
    restart: always
  elasticsearch3:
    image: harbor.dockerregistry.com/app/elasticsearch:7.17.1
    container_name: elasticsearch3
    environment:
      - node.name=elasticsearch3
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - discovery.seed_hosts=elasticsearch,elasticsearch2
      - cluster.initial_master_nodes=elasticsearch,elasticsearch2,elasticsearch3
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /data/es/data03:/usr/share/elasticsearch/data
    networks:
      - esnet
    restart: always

volumes:
  data01:
  data02:
  data03:

networks:
  esnet:
```