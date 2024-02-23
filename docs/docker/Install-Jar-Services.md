## Compose file

```yaml
version: '3'
x-service-template: &x-service-template
  restart: always
  extra_hosts:
    - "nacos-dev:192.168.1.12"
  environment:
    SPRING_PROFILES_ACTIVE: dev
    SPRING_CLOUD_NACOS_DISCOVERY_SERVER_ADDR: nacos-dev:8848
    SPRING_CLOUD_NACOS_CONFIG_SERVER_ADDR: nacos-dev:8848
    SPRING_CLOUD_NACOS_USERNAME: nacos
    SPRING_CLOUD_NACOS_PASSWORD: nacos
  volumes:
    - /etc/localtime:/etc/localtime:ro
  networks:
  - app

services:
  onemap:
    <<: *x-service-template
    build: ./onemap
    container_name: onemap
    ports:
      - 8080:8080

networks:
  app:
    external: false
```





## Dir tree

```shell
$ ls  -lh
total 182M
-rw-r--r-- 1 PC 197121  619 Dec  8 10:47 docker-compose.yaml
-rw-r--r-- 1 PC 197121 182M Dec  8 10:32 jdk.tar
drwxr-xr-x 1 PC 197121    0 Dec  8 10:44 onemap/


$ ls  -lh onemap/
total 1.0K
-rw-r--r-- 1 PC 197121 615 Dec  8 10:37 Dockerfile
drwxr-xr-x 1 PC 197121   0 Dec  8 10:44 target/


$ ls  -lh onemap/target/
total 1.0K
-rw-r--r-- 1 PC 197121 240 Dec  8 10:43 onemap.jar
```





## Dockerfile

```Dockerfile
FROM harbor.dockerregistry.com/app/adoptopenjdk/openjdk8-openj9:alpine-slim-local2

WORKDIR /home
ADD ./target/*.jar ./app.jar

#RUN useradd -ms /bin/bash appuser
#USER appuser

CMD java -Djava.security.egd=file:/dev/./urandom -jar app.jar \
    --spring.profiles.active=$SPRING_PROFILES_ACTIVE \
    --spring.cloud.nacos.discovery.server-addr=$SPRING_CLOUD_NACOS_DISCOVERY_SERVER_ADDR \
    --spring.cloud.nacos.config.server-addr=$SPRING_CLOUD_NACOS_CONFIG_SERVER_ADDR \
    --spring.cloud.nacos.username=$SPRING_CLOUD_NACOS_USERNAME \
    --spring.cloud.nacos.password=$SPRING_CLOUD_NACOS_PASSWORD
```









## Start Service

```shell
export COMPOSE_DOCKER_CLI_BUILD=0
export DOCKER_BUILDKIT=0

docker-compose up # 测试
docker-compose down  # 清理
docker-compose up -d # 启动

docker login -u guest -p Guest@123.com harbor.dockerregistry.com
docker logout harbor.dockerregistry.com # /root/.docker/config.json
```

