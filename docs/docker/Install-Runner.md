## 1 Compose File



```yaml
version: '3'
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    privileged: true
    restart: always
    volumes:
      - ./gitlab-runner/:/etc/gitlab-runner      
      - ~/.docker:/root/.docker
      - /var/run/docker.sock:/var/run/docker.sock
      #- ./cache:/cache
```



## 2 Register Runner

```shell
root@DEVOPS:~/gitlab-runner-npm# cat auto-register-runner.sh 
#!/bin/bash

REGISTRATION_TOKEN="GR13489413zMeyMkh3zq7nNsvaPZe"
RUNNER_NAME="gitlab-runner-npm-group"

docker exec -it "${RUNNER_NAME}" gitlab-runner register \
  --non-interactive \
  --url "http://172.31.94.6/" \
  --registration-token "${REGISTRATION_TOKEN}" \
  --executor "docker" \
  --docker-image "docker:19.03.12" \
  --description "Docker Runner" \
  --tag-list "docker-npm" \
  --locked=true \
  --docker-privileged=true \
  --run-untagged=false \
  --docker-tlsverify=false \
  --docker-disable-entrypoint-overwrite=false \
  --docker-oom-kill-disable=false \
  --docker-disable-cache=false \
  --docker-shm-size=0 \
  --cache-type="s3" \
  --cache-path="runner" \
  --cache-shared=true \
  --cache-s3-server-address="192.1.2.100:9000" \
  --cache-s3-access-key="08JbSVlKWEYc1fc8xFwf" \
  --cache-s3-secret-key="JICzCcjqXTa5DVk6ZIIOFWBafZv32HFYV4arNUD4" \
  --cache-s3-bucket-name="runner-cache-papd" \
  --cache-s3-insecure=true
```





## 3 Runner Config	

```shell
root@DEVOPS:~/gitlab-runner-npm# cat gitlab-runner/config.toml 
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "Docker Runner"
  url = "http://172.31.94.6/"
  token = "y_L4CRYByzvxbTxiryFJ"
  executor = "docker"
  [runners.custom_build_dir]
  #[runners.cache]
  #  Type = "local"
  #  Path = "/cache"
  #  Shared = true
  [runners.cache]
    Type = "s3"
    Path = "runner"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "192.1.2.100:9000"
      AccessKey = "08JbSVlKWEYc1fc8xFwf"
      SecretKey = "JICzCcjqXTa5DVk6ZIIOFWBafZv32HFYV4arNUD4"
      BucketName = "runner-cache-papd"
      Insecure = true
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker:19.03.12"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 536870912
    memory = "24000m"
    memory_swap = "26g"
    #memory_reservation = "8000m"
    cpus = "12"
    allowed_images = [
      "harbor.dockerregistry.com/kubernetes/maven:3.6.3-openjdk-8-slim",
      "harbor.dockerregistry.com/kubernetes/kaniko-project/executor:v1.14.0-debug",
      "harbor.dockerregistry.com/kubernetes/argocli-git:v2.7.14",
      "harbor.dockerregistry.com/kubernetes/nodejs:*"
    ]
```

