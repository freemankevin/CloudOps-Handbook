## Deamon.json

https://docs.docker.com/engine/reference/commandline/dockerd/#daemon

```json
{
  "registry-mirrors": ["https://ustc-edu-cn.mirror.aliyuncs.com"],
  "debug": false,
  "insecure-registries": [
    "0.0.0.0/0"
  ],
  "ip-forward": true,
  "ipv6": false,
  "live-restore": true,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "100m",
    "max-file": "2"
  },
  "selinux-enabled": false,
  "experimental" : true,
  "storage-driver": "overlay2",
  "metrics-addr" : "0.0.0.0:9323",
  "data-root": "/data/docker",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```



## config.json

```json
// /root/.docker/config.json
// base64加密
{
        "auths": {
                "harbor.dockerregistry.com": {
                        "auth": "YWRtaW46SGFyYm9yMTIzNDU="
                }
        }
}
```





## Dockefile

> Clean up the mirror environment

### ubuntu

```shell
RUN apt-get update && \
    apt-get install -y --no-install-recommends git && \
    rm -rf /tmp/* && \
    rm -rf /var/lib/apt/lists/*
```

### alpine

```Dockerfile
RUN apk -U --no-cache add git
# 或者
RUN apk -U add git && \
    rm -rf /var/cache/apk/*
```



### Backend

```dockerfile
# Define arguments for host, JDK version
ARG HARBOR_HOST
ARG JDK_VERSION=8-latest

# Use arguments in the FROM statement for the JDK version
FROM ${HARBOR_HOST}/library/azul/zulu-openjdk-alpine:${JDK_VERSION}

# Define arguments for application version, port, and jar name
ARG APP_VERSION=5.1.0
ARG APP_PORT=8510
ARG JAR_NAME=sys-gateway

WORKDIR /home

# Copy the jar file using the argument for the version and jar name
COPY ./target/${JAR_NAME}-${APP_VERSION}.jar ./app.jar

# Expose the port using the argument
EXPOSE ${APP_PORT}

# Set environment variable for port, this could be used at runtime if the application reads from this environment variable
ENV HTTPS_PORT=${APP_PORT}
ENV HTTP_PORT=${APP_PORT}

# Set the CMD or ENTRYPOINT using the environment variables
CMD java -Djava.security.egd=file:/dev/./urandom -jar app.jar \
    --server.port=${HTTPS_PORT} \
    --http.port=${HTTP_PORT} \
    --spring.profiles.active=$SPRING_PROFILES_ACTIVE \
    --spring.cloud.nacos.discovery.server-addr=$SPRING_CLOUD_NACOS_DISCOVERY_SERVER_ADDR \
    --spring.cloud.nacos.config.server-addr=$SPRING_CLOUD_NACOS_CONFIG_SERVER_ADDR \
    --spring.cloud.nacos.username=$SPRING_CLOUD_NACOS_USERNAME \
    --spring.cloud.nacos.password=$SPRING_CLOUD_NACOS_PASSWORD \
    --spring.cloud.nacos.config.namespace=$SPRING_CLOUD_NACOS_CONFIG_NAMESPACE \
    --spring.cloud.nacos.discovery.namespace=$SPRING_CLOUD_NACOS_DISCOVERY_NAMESPACE
```



### Frontend

```dockerfile
ARG HARBOR_HOST
ARG NG_VERSION=stable-alpine3.17
FROM ${HARBOR_HOST}/library/nginx:${NG_VERSION}
#RUN  mkdir -p /usr/share/nginx/html/dir
COPY dist /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

