## 1 Pull runner

```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update gitlab
helm search repo -l gitlab/gitlab-runner | grep 15.11


cd ~
mkdir -p gitlab-runner && cd gitlab-runner
helm pull gitlab/gitlab-runner --version v0.52.1
tar xf gitlab-runner-0.52.1.tgz 
cp gitlab-runner/values.yaml{,.bak} 


root@master1:~/gitlab-runner# ll 
total 28
drwxr-xr-x 4 root root  4096 Dec  7 10:01 gitlab-runner
-rw-r--r-- 1 root root 23737 Dec  7 10:01 gitlab-runner-0.52.1.tgz
root@master1:~/gitlab-runner# ll gitlab-runner
total 100
-rw-r--r-- 1 root root 14127 Jun  2  2023 CHANGELOG.md
-rw-r--r-- 1 root root   411 Jun  2  2023 Chart.yaml
-rw-r--r-- 1 root root   796 Jun  2  2023 CONTRIBUTING.md
-rw-r--r-- 1 root root  1084 Jun  2  2023 LICENSE
-rw-r--r-- 1 root root   917 Jun  2  2023 Makefile
-rw-r--r-- 1 root root  1305 Jun  2  2023 NOTICE
-rw-r--r-- 1 root root   219 Jun  2  2023 README.md
drwxr-xr-x 2 root root  4096 Dec  7 10:01 templates
-rw-r--r-- 1 root root 26508 Jun  2  2023 values.yaml
-rw-r--r-- 1 root root 26508 Dec  7 10:01 values.yaml.bak
```



## 2 Edit config

```bash
# 编辑配置
vim gitlab-runner/values.yaml
### 
image:
  registry: docker.io
  image: gitlab/gitlab-runner
  tag: alpine-v15.11.1
  
imagePullSecrets:
  - name: "harbor-credentials"

replicas: 1

gitlabUrl: http://10.0.1.1/ ## gitlab http
#certsSecretName: runner-tls-chain   ## gitlab https need.

concurrent: 10

logLevel: info

rbac:
  create: true

metrics:
  enabled: true
  portName: metrics
  port: 9252
  serviceMonitor:
    enabled: false

runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:16.04"
      [runners.custom_build_dir]
        enabled = true
      [runners.cache]
        Type = "s3"
        Path = "runner"
        Shared = true
        [runners.cache.s3]
          ServerAddress = "192.168.1.120:9000"
          BucketName = "runner-cache-papd"
          AccessKey = "4IABo3LtOGfc8QbW5eEY"
          SecretKey = "7mc1JtsWRjuD6Mk1pxvAXMRSdwunT7Jq7UURAFmY"
          #BucketLocation = ""
          Insecure = true
          #AuthenticationType = "access-key"  

  executor: kubernetes
  privileged: true
  
  tags: "kubernetes"
  
  secret: gitlab-runner

  builds: 
    cpuLimit: 2010m
    cpuLimitOverwriteMaxAllowed: 2010m
    memoryLimit: 2060Mi
    memoryLimitOverwriteMaxAllowed: 2060Mi
    cpuRequests: 100m
    cpuRequestsOverwriteMaxAllowed: 100m
    memoryRequests: 128Mi
    memoryRequestsOverwriteMaxAllowed: 128Mi

  services: 
    cpuLimit: 200m
    memoryLimit: 256Mi
    cpuRequests: 100m
    memoryRequests: 128Mi

  helpers:
    cpuLimit: 200m
    memoryLimit: 256Mi
    cpuRequests: 100m
    memoryRequests: 128Mi
    # image: "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:x86_64-${CI_RUNNER_REVISION}"
    image: "harbor.dockerregistry.com/kubernetes/gitlab/gitlab-runner-helper:x86_64-v15.11.1"


  resources: 
    limits:
      memory: 256Mi
      cpu: 200m
    requests:
      memory: 128Mi
      cpu: 100m
```



## 3 Create secret

```bash
# 创建命名空间
kubectl create ns gitlab-runner-papd

# 创建密钥
# 创建拉取镜像的认证秘钥
# 这里使用的是Robot 账户
kubectl create secret docker-registry harbor-credentials \
    --docker-server=harbor.dockerregistry.com \
    --docker-username=robot\$5m2wwklm4tet \
    --docker-password=Lf1Gpxz9zfVavMJk2xPFzBdRYE2TIxGG \
    -n gitlab-runner-papd 
    
# 创建连接Gitlab的认证秘钥  ## gitlab https
# kubectl create secret generic mygitlab-mydomain-com-crt   --namespace gitlab-runner-papd   --from-file=mygitlab.mydomain.com.crt=mygitlab.mydomain.com.crt


# 使用Gitlab 项目或项目组的Token，来创建Runner注册到Gitlab的秘钥
kubectl create secret generic gitlab-runner \
  --from-literal=runner-registration-token=GR1348941Bdbk4zQ3TxSpCNKzysSe \
  --from-literal=runner-token="" \
  --type=Opaque \
  -n gitlab-runner-papd
```



## 4 Install Runner

```bash
# 安装


# helm install --namespace <NAMESPACE> gitlab-runner -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner --version <RUNNER_HELM_CHART_VERSION>
# helm search repo -l gitlab/gitlab-runner | grep 15.11
# helm install gitlab-runner-papd -f gitlab-runner/values.yaml gitlab/gitlab-runner  --namespace gitlab-runner-papd --create-namespace

helm install gitlab-runner-papd ./gitlab-runner -f gitlab-runner/values.yaml   --namespace gitlab-runner-papd --create-namespace

```



## 5 Update Runner

```bash
# 更新配置，更新服务
helm upgrade gitlab-runner-papd  ./gitlab-runner -f gitlab-runner/values.yaml   --namespace gitlab-runner-papd

# 完全卸载
helm -n gitlab-runner-papd uninstall gitlab-runner
```



## 6 ADD Network Support

```bash
## 1 修改主配置并重启服务：

# 主配置修改
$ vim /etc/gitlab/gitlab.rb
gitlab_rails['outbound_local_requests'] = { "allow" => true }

# 重启服务
gitlab-ctl restart


## 2 页面配置修改，并添加白名单：

# 修改入口：
http(s)://<gitlab-server.addr>:<server-port>/admin/application_settings/network
# 勾选上：
- [x] Allow requests to the local network from webhooks and integrations
- [x] Allow requests to the local network from system hooks
# 添加白名单：
## 添加自己要访问的内网域名或IP地址
** Local IP addresses and domain names that hooks and integrations can access ** 
harbor.dockerregistry.com
minio.objectstorage.com
traefik.k8scluster.com
argocd.k8scluster.com
grpc.argocd.k8scluster.com
192.168.1.120
```





## 7 ADD Harbor Support

```bash
# 页面上配置Harbor 集成
http(s)://<gitlab-server.addr>:<server-port>/groups/cicd/-/settings/integrations

## 找到Harbor

- [x] Enable integration

**Harbor URL**
https://harbor.dockerregistry.com

**Harbor project name**
papd ## 写自己项目镜像的 Harbor 项目名

**Harbor username**
robot$5m2wwklm4tet

**Enter new Harbor password**
Lf1Gpxz9zfVavMJk2xPFzBdRYE2TIxGG  ## 写自己Harbor 那边机器人账户、密码
```



## 8 Copy Harbor ca

```shell
# Harbor 镜像拉取时证书认证报错：
# Running with gitlab-runner 15.11.1 (0d8a024e)
#   on gitlab-runner-papd-f64fd85f5-zh599 E5CnU17q, system ID: r_3yoDCGyFCkI7
# Preparing the "kubernetes" executor
# 00:00
# Using Kubernetes namespace: gitlab-runner-papd
# Using Kubernetes executor with image harbor.dockerregistry.com/kubernetes/node:16.20.2 ...
# Using attach strategy to execute scripts...
# Preparing environment
# 00:03
# Waiting for pod gitlab-runner-papd/runner-e5cnu17q-project-3013-concurrent-0j8wls to be running, status is Pending
# WARNING: Failed to pull image with policy "": image pull failed: rpc error: code = Unknown desc = failed to pull and unpack image \
# "harbor.dockerregistry.com/kubernetes/node:16.20.2": failed to resolve reference "harbor.dockerregistry.com/kubernetes/node:16.20.2": \
# failed to do request: Head "https://harbor.dockerregistry.com/v2/kubernetes/node/manifests/16.20.2": tls: failed to verify certificate: x509: certificate signed by unknown authority
# ERROR: Job failed: prepare environment: waiting for pod running: pulling image "harbor.dockerregistry.com/kubernetes/node:16.20.2": image pull failed: rpc error: \
# code = Unknown desc = failed to pull and unpack image "harbor.dockerregistry.com/kubernetes/node:16.20.2": failed to resolve reference "harbor.dockerregistry.com/kubernetes/node:16.20.2":\
# failed to do request: Head "https://harbor.dockerregistry.com/v2/kubernetes/node/manifests/16.20.2": \
# tls: failed to verify certificate: x509: certificate signed by unknown authority. Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more information


# on worker nodes.

\cp -rvf /etc/tls/harbor/harbor.dockerregistry.com/ca.crt /etc/ssl/certs/
\cp -rvf /etc/tls/harbor/harbor.dockerregistry.com/harbor.dockerregistry.com.cert /etc/ssl/certs/

# 更新证书存储
update-ca-certificates

# 重启 运行时 服务
systemctl restart containerd
```



## 9 Test Limit

> 数据已脱敏。



可以看到相关pod 环境均按开始`values.yaml` 资源限制来启动的

```shell
root@master2:~/gitlab-runner# kubectl get pod -n gitlab-runner-papd runner-xdnsadt3-project-2806-concurrent-0rg4lh   -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/containerID: xxxxx
    cni.projectcalico.org/podIP: xxxxx
    cni.projectcalico.org/podIPs: xxxxx
    job.runner.gitlab.com/before_sha: xxxxx
    job.runner.gitlab.com/id: "xxxxx"
    job.runner.gitlab.com/name: package
    job.runner.gitlab.com/ref: test
    job.runner.gitlab.com/sha: xxxxx
    job.runner.gitlab.com/url: http://xxxxx/frame-ui/-/jobs/16326
    project.runner.gitlab.com/id: "xxxx"
  creationTimestamp: "2024-01-24T08:05:45Z"
  generateName: runner-xdnsadt3-project-2806-concurrent-0
  labels:
    pod: runner-xdnsadt3-project-2806-concurrent-0
  name: runner-xdnsadt3-project-2806-concurrent-0rg4lh
  namespace: gitlab-runner-papd
  resourceVersion: "3582460"
  uid: 8b8299ff-1758-4e68-9fb5-4a6d81827f08
spec:
  affinity: {}
  containers:
  - command:
    - sh
    - -c
    - "if [ -x /usr/local/bin/bash ]; then\n\texec /usr/local/bin/bash \nelif [ -x
      /usr/bin/bash ]; then\n\texec /usr/bin/bash \nelif [ -x /bin/bash ]; then\n\texec
      /bin/bash \nelif [ -x /usr/local/bin/sh ]; then\n\texec /usr/local/bin/sh \nelif
      [ -x /usr/bin/sh ]; then\n\texec /usr/bin/sh \nelif [ -x /bin/sh ]; then\n\texec
      /bin/sh \nelif [ -x /busybox/sh ]; then\n\texec /busybox/sh \nelse\n\techo shell
      not found\n\texit 1\nfi\n\n"
    env:
    - name: FF_CMD_DISABLE_DELAYED_ERROR_LEVEL_EXPANSION
      value: "false"
    - name: FF_NETWORK_PER_BUILD
      value: "false"
    - name: FF_USE_LEGACY_KUBERNETES_EXECUTION_STRATEGY
      value: "false"
    - name: FF_USE_DIRECT_DOWNLOAD
      value: "true"
    - name: FF_SKIP_NOOP_BUILD_STAGES
      value: "true"
    - name: FF_USE_FASTZIP
      value: "false"
    - name: FF_DISABLE_UMASK_FOR_DOCKER_EXECUTOR
      value: "false"
    - name: FF_ENABLE_BASH_EXIT_CODE_CHECK
      value: "false"
    - name: FF_USE_WINDOWS_LEGACY_PROCESS_STRATEGY
      value: "true"
    - name: FF_USE_NEW_BASH_EVAL_STRATEGY
      value: "false"
    - name: FF_USE_POWERSHELL_PATH_RESOLVER
      value: "false"
    - name: FF_USE_DYNAMIC_TRACE_FORCE_SEND_INTERVAL
      value: "false"
    - name: FF_SCRIPT_SECTIONS
      value: "false"
    - name: FF_USE_NEW_SHELL_ESCAPE
      value: "false"
    - name: FF_ENABLE_JOB_CLEANUP
      value: "false"
    - name: FF_KUBERNETES_HONOR_ENTRYPOINT
      value: "false"
    - name: FF_POSIXLY_CORRECT_ESCAPES
      value: "false"
    - name: FF_USE_IMPROVED_URL_MASKING
      value: "false"
    - name: FF_RESOLVE_FULL_TLS_CHAIN
      value: "true"
    - name: FF_DISABLE_POWERSHELL_STDIN
      value: "false"
    - name: FF_USE_POD_ACTIVE_DEADLINE_SECONDS
      value: "false"
    - name: FF_USE_ADVANCED_POD_SPEC_CONFIGURATION
      value: "false"
    - name: FF_SET_PERMISSIONS_BEFORE_CLEANUP
      value: "true"
    - name: CI_JOB_IMAGE
      value: xxxxxxxxxxxxxxx/kubernetes/nodejs:v4
    - name: CI_RUNNER_SHORT_TOKEN
      value: xDnSadT3
    - name: CI_BUILDS_DIR
      value: /builds
    - name: CI_PROJECT_DIR
      value: /builds/rdcenter/apply/product/productivity/operation/source/frame-ui
    - name: CI_CONCURRENT_ID
      value: "0"
    - name: CI_CONCURRENT_PROJECT_ID
      value: "0"
    - name: CI_SERVER
      value: "yes"
    - name: CI_JOB_STATUS
      value: running
    - name: CI_JOB_TIMEOUT
      value: "3600"
    - name: CI_PIPELINE_ID
      value: "4944"
    - name: CI_PIPELINE_URL
      value: http://xxxxxxxxxxxxxx/frame-ui/-/pipelines/4944
    - name: CI_JOB_ID
      value: "16326"
    - name: CI_JOB_URL
      value: http://xxxxxxxxxxxxxx/frame-ui/-/jobs/16326
    - name: CI_JOB_STARTED_AT
      value: "2024-01-24T08:05:45Z"
    - name: CI_BUILD_ID
      value: "16326"
    - name: CI_REGISTRY_USER
      value: gitlab-ci-token
    - name: HARBOR_URL
      value: https://xxxxxxxxxxxxxxx
    - name: HARBOR_HOST
      value: xxxxxxxxxxxxxxx
    - name: HARBOR_OCI
      value: oci://xxxxxxxxxxxxxxx
    - name: HARBOR_PROJECT
      value: papd
    - name: HARBOR_USERNAME
      value: robot$5m2wwklm4tet
    - name: CI_DEPENDENCY_PROXY_USER
      value: gitlab-ci-token
    - name: CI_JOB_NAME
      value: package
    - name: CI_JOB_NAME_SLUG
      value: package
    - name: CI_JOB_STAGE
      value: package
    - name: CI_JOB_MANUAL
      value: "true"
    - name: CI_NODE_TOTAL
      value: "1"
    - name: CI_BUILD_NAME
      value: package
    - name: CI_BUILD_STAGE
      value: package
    - name: CI_BUILD_MANUAL
      value: "true"
    - name: CI
      value: "true"
    - name: GITLAB_CI
      value: "true"
    - name: CI_SERVER_URL
      value: http://xxxxxxxxxxxxxxxxxxxx
    - name: CI_SERVER_HOST
      value: xxxxxxxxxxx
    - name: CI_SERVER_PORT
      value: "xxxxx"
    - name: CI_SERVER_PROTOCOL
      value: http
    - name: CI_SERVER_SHELL_SSH_HOST
      value: xxxxxxxxxxx
    - name: CI_SERVER_SHELL_SSH_PORT
      value: "22"
    - name: CI_SERVER_NAME
      value: GitLab
    - name: CI_SERVER_VERSION
      value: 15.11.13
    - name: CI_SERVER_VERSION_MAJOR
      value: "15"
    - name: CI_SERVER_VERSION_MINOR
      value: "11"
    - name: CI_SERVER_VERSION_PATCH
      value: "13"
    - name: CI_SERVER_REVISION
      value: cc3748fcd2d
    - name: GITLAB_FEATURES
    - name: CI_PROJECT_ID
      value: "2806"
    - name: CI_PROJECT_NAME
      value: frame-ui
    - name: CI_PROJECT_TITLE
      value: frame-ui
    - name: CI_PROJECT_DESCRIPTION
      value: xxxxxx
    - name: CI_PROJECT_PATH
      value: rdcenter/apply/product/productivity/operation/source/frame-ui
    - name: CI_PROJECT_PATH_SLUG
      value: rdcenter-apply-product-productivity-operation-source-frame-ui
    - name: CI_PROJECT_NAMESPACE
      value: rdcenter/apply/product/productivity/operation/source
    - name: CI_PROJECT_NAMESPACE_ID
      value: "1996"
    - name: CI_PROJECT_ROOT_NAMESPACE
      value: rdcenter
    - name: CI_PROJECT_URL
      value: http://xxxxxxxxxxxxxx/frame-ui
    - name: CI_PROJECT_VISIBILITY
      value: private
    - name: CI_PROJECT_REPOSITORY_LANGUAGES
      value: html,javascript,css,typescript,vue
    - name: CI_PROJECT_CLASSIFICATION_LABEL
    - name: CI_DEFAULT_BRANCH
      value: xxxxxx
    - name: CI_CONFIG_PATH
      value: .gitlab-ci.yml
    - name: CI_PAGES_DOMAIN
      value: xxxxxxx
    - name: CI_PAGES_URL
      value: http://xxxxxx/frame-ui
    - name: CI_DEPENDENCY_PROXY_SERVER
      value: xxxxxxxxxxxxxxxxxxxx
    - name: CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX
      value: xxxxxxxxxxxxxxxxxxxx/rdcenter/dependency_proxy/containers
    - name: CI_DEPENDENCY_PROXY_DIRECT_GROUP_IMAGE_PREFIX
      value: xxxxxxxxxxxxxx/dependency_proxy/containers
    - name: CI_API_V4_URL
      value: http://xxxxxxxxxxxxxxxxxxxx/api/v4
    - name: CI_API_GRAPHQL_URL
      value: http://xxxxxxxxxxxxxxxxxxxx/api/graphql
    - name: CI_TEMPLATE_REGISTRY_HOST
      value: registry.gitlab.com
    - name: CI_PIPELINE_IID
      value: "182"
    - name: CI_PIPELINE_SOURCE
      value: push
    - name: CI_PIPELINE_CREATED_AT
      value: "2024-01-17T07:46:52Z"
    - name: CI_COMMIT_SHA
      value: 96e25221890613845bc2a6a14de49add95090248
    - name: CI_COMMIT_SHORT_SHA
      value: 96e25221
    - name: CI_COMMIT_BEFORE_SHA
      value: a71843f930b1a993cc56dca99d1de51418ca3600
    - name: CI_COMMIT_REF_NAME
      value: test
    - name: CI_COMMIT_REF_SLUG
      value: test
    - name: CI_COMMIT_BRANCH
      value: test
    - name: CI_COMMIT_MESSAGE
      value: |
        ci: update template
    - name: CI_COMMIT_TITLE
      value: 'ci: update template'
    - name: CI_COMMIT_DESCRIPTION
    - name: CI_COMMIT_REF_PROTECTED
      value: "false"
    - name: CI_COMMIT_TIMESTAMP
      value: "2024-01-17T15:46:34+08:00"
    - name: CI_COMMIT_AUTHOR
      value: xxxxxxxxx <xxxxxxxxx>
    - name: CI_BUILD_REF
      value: 96e25221890613845bc2a6a14de49add95090xxxx
    - name: CI_BUILD_BEFORE_SHA
      value: a71843f930b1a993cc56dca99d1de51418ca3600
    - name: CI_BUILD_REF_NAME
      value: test
    - name: CI_BUILD_REF_SLUG
      value: test
    - name: CI_RUNNER_ID
      value: "43"
    - name: CI_RUNNER_DESCRIPTION
      value: gitlab-runner-papd-89cd578d4-cqxc7
    - name: CI_RUNNER_TAGS
      value: '["kubernetes"]'
    - name: DEPLOY_ONLY
      value: "false"
    - name: REPO_URL
      value: http://gitlab-ci-token:xxxxxx@xxxxxxxxxxxxxx/frame-ui.git
    - name: DEST_ENVIRONMENT
      value: prod
    - name: CI_PROJECT_NAME
      value: frame-ui
    - name: HARBOR_PROJECT
      value: papd-bom
    - name: GITLAB_USER_ID
      value: "175"
    - name: GITLAB_USER_EMAIL
      value: xxxxxx
    - name: GITLAB_USER_LOGIN
      value: xxxxxxxxxx
    - name: GITLAB_USER_NAME
      value: xxxxxx
    - name: CI_DISPOSABLE_ENVIRONMENT
      value: "true"
    - name: CI_RUNNER_VERSION
      value: 15.11.1
    - name: CI_RUNNER_REVISION
      value: 0d8a024e
    - name: CI_RUNNER_EXECUTABLE_ARCH
      value: linux/amd64
    - name: RUNNER_TEMP_PROJECT_DIR
      value: /builds/rdcenter/apply/product/productivity/operation/source/frame-ui.tmp
    image: xxxxxxxxxxxxxxx/kubernetes/nodejs:v4
    imagePullPolicy: IfNotPresent
    name: build
    resources:
      limits:
        cpu: 2010m
        memory: 2060Mi
      requests:
        cpu: 100m
        memory: 128Mi
    securityContext:
      capabilities:
        drop:
        - NET_RAW
      privileged: true
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /scripts-2806-16326
      name: scripts
    - mountPath: /logs-2806-16326
      name: logs
    - mountPath: /builds
      name: repo
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-4xzsk
      readOnly: true
  - command:
    - sh
    - -c
    - "if [ -x /usr/local/bin/bash ]; then\n\texec /usr/local/bin/bash \nelif [ -x
      /usr/bin/bash ]; then\n\texec /usr/bin/bash \nelif [ -x /bin/bash ]; then\n\texec
      /bin/bash \nelif [ -x /usr/local/bin/sh ]; then\n\texec /usr/local/bin/sh \nelif
      [ -x /usr/bin/sh ]; then\n\texec /usr/bin/sh \nelif [ -x /bin/sh ]; then\n\texec
      /bin/sh \nelif [ -x /busybox/sh ]; then\n\texec /busybox/sh \nelse\n\techo shell
      not found\n\texit 1\nfi\n\n"
    image: xxxxxxxxxxxxxxx/kubernetes/gitlab/gitlab-runner-helper:x86_64-v15.11.1
    imagePullPolicy: IfNotPresent
    name: helper
    resources:
      limits:
        cpu: 200m
        memory: 256Mi
      requests:
        cpu: 100m
        memory: 128Mi
    securityContext:
      capabilities:
        drop:
        - NET_RAW
      privileged: true
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /scripts-2806-16326
      name: scripts
    - mountPath: /logs-2806-16326
      name: logs
    - mountPath: /builds
      name: repo
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-4xzsk
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  imagePullSecrets:
  - name: runner-xdnsadt3-project-2806-concurrent-0mqh6n
  initContainers:
  - command:
    - sh
    - -c
    - touch /logs-2806-16326/output.log && (chmod 777 /logs-2806-16326/output.log
      || exit 0)
    image: xxxxxxxxxxxxxxx/kubernetes/gitlab/gitlab-runner-helper:x86_64-v15.11.1
    imagePullPolicy: IfNotPresent
    name: init-permissions
    resources:
      limits:
        cpu: 200m
        memory: 256Mi
      requests:
        cpu: 100m
        memory: 128Mi
    securityContext:
      capabilities:
        drop:
        - NET_RAW
      privileged: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /scripts-2806-16326
      name: scripts
    - mountPath: /logs-2806-16326
      name: logs
    - mountPath: /builds
      name: repo
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-4xzsk
      readOnly: true
  nodeName: worker1.k8scluster.com
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Never
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 0
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - emptyDir: {}
    name: repo
  - emptyDir: {}
    name: scripts
  - emptyDir: {}
    name: logs
  - name: kube-api-access-4xzsk
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-01-24T08:05:46Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2024-01-24T08:05:48Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2024-01-24T08:05:48Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2024-01-24T08:05:45Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://bb6ca8456a54c2ee8c72bb384f9e7c3c896763a0406e07be6adc11064ec7e8fc
    image: xxxxxxxxxxxxxxx/kubernetes/nodejs:v4
    imageID: sha256:3b30b0ab418c68619187e9c61a30afa8309cb81145def1ab4e08c7962497ffba
    lastState: {}
    name: build
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2024-01-24T08:05:47Z"
  - containerID: containerd://bda6ee9607dae7303fe15dd05e6e0d357477484020dce00d4c8e240c0aac9fda
    image: xxxxxxxxxxxxxxx/kubernetes/gitlab/gitlab-runner-helper:x86_64-v15.11.1
    imageID: sha256:61a243bb78908ae5daf5570de71c972ef2a5706fda848c137f41b7534082654f
    lastState: {}
    name: helper
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2024-01-24T08:05:47Z"
  hostIP: xxxxxxx
  initContainerStatuses:
  - containerID: containerd://9170fc749cc1383cf142b3c6371bf1e889aeef95d2f3cca0ea4327bf02e65eb4
    image: xxxxxxxxxxxxxxx/kubernetes/gitlab/gitlab-runner-helper:x86_64-v15.11.1
    imageID: sha256:61a243bb78908ae5daf5570de71c972ef2a5706fda848c137f41b7534082654f
    lastState: {}
    name: init-permissions
    ready: true
    restartCount: 0
    state:
      terminated:
        containerID: containerd://9170fc749cc1383cf142b3c6371bf1e889aeef95d2f3cca0ea4327bf02e65eb4
        exitCode: 0
        finishedAt: "2024-01-24T08:05:46Z"
        reason: Completed
        startedAt: "2024-01-24T08:05:46Z"
  phase: Running
  podIP: 
  podIPs:
  - ip: 
  qosClass: Burstable
  startTime: "2024-01-24T08:05:45Z"
```