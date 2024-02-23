## 1 版本适配

[官方文档](https://developer.hashicorp.com/consul/tutorials/kubernetes/kubernetes-deployment-guide#configure-kubernetes-secrets)


> docker 版本可能也需要考虑，暂未遇到

>已成功部署 `v1.0.6 、v1.1.1`，其他可能有坑

| `helmRepo`         | `chartVersion` | `appVersion` | `kubeVersion` |
|--------------------|----------------|--------------|---------------|
| `hashicorp/consul` | 1.1.1          | 1.15.1       | >=1.22.0-0    |
| `hashicorp/consul` | 1.1.0          | 1.15.0       | >=1.22.0-0    |
| `hashicorp/consul` | 1.0.6          | 1.14.5       | >=1.21.0-0    |
| `hashicorp/consul` | 1.0.5          | 1.14.5       | >=1.21.0-0    |
| `hashicorp/consul` | 1.0.4          | 1.14.4       | >=1.21.0-0    |
| `hashicorp/consul` | 1.0.3          | 1.14.4       | >=1.21.0-0    |
| `hashicorp/consul` | 1.0.2          | 1.14.2       | >=1.21.0-0    |
| `hashicorp/consul` | 1.0.1          | 1.14.1       | >=1.21.0-0    |
| `hashicorp/consul` | 1.0.0          | 1.14.0       | >=1.21.0-0    |
| `hashicorp/consul` | 0.49.5         | 1.13.7       | >=1.21.0-0    |
| `hashicorp/consul` | 0.49.4         | 1.13.6       | >=1.21.0-0    |
| `hashicorp/consul` | 0.49.3         | 1.13.6       | >=1.21.0-0    |
| `hashicorp/consul` | 0.49.2         | 1.13.4       | >=1.21.0-0    |
| `hashicorp/consul` | 0.49.1         | 1.13.3       | >=1.21.0-0    |
| `hashicorp/consul` | 0.49.0         | 1.13.2       | >=1.21.0-0    |
| `hashicorp/consul` | 0.48.0         | 1.13.1       | >=1.21.0-0    |
| `hashicorp/consul` | 0.47.1         | 1.13.1       | >=1.19.0-0    |
| `hashicorp/consul` | 0.47.0         | 1.13.1       | >=1.19.0-0    |
| `hashicorp/consul` | 0.46.1         | 1.12.3       | >=1.19.0-0    |
| `hashicorp/consul` | 0.46.0         | 1.12.3       | >=1.19.0-0    |
| `hashicorp/consul` | 0.45.0         | 1.12.2       | >=1.19.0-0    |
| `hashicorp/consul` | 0.44.0         | 1.12.0       | >=1.19.0-0    |
| `hashicorp/consul` | 0.43.0         | 1.12.0       | >=1.19.0-0    |
| `hashicorp/consul` | 0.42.0         | 1.11.4       | >=1.19.0-0    |
| `hashicorp/consul` | 0.41.1         | 1.11.3       | >=1.18.0-0    |
| `hashicorp/consul` | 0.41.0         | 1.11.3       | >=1.18.0-0    |



## 2 Helm部署

```shell
# 添加helm仓库
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# 在线部署
vim config.yaml
```

>`imageConsulDataplane`  在低版本中可能不存在，需要将这行注解掉，否则可能无法部署成功

```yaml
global:
  name: consul
  image: "harbor.dockerregistry.com/gitops/hashicorp/consul:1.14.5"
  imageK8S: harbor.dockerregistry.com/gitops/hashicorp/consul-k8s-control-plane:1.0.6
  gossipEncryption:
    # Automatically generate a gossip encryption key and save it to a Kubernetes or Vault secret.
    autoGenerate: true
  tls:
    # If true, the Helm chart will enable TLS for Consul
    # servers and clients and all consul-k8s-control-plane components, as well as generate certificate
    # authority (optional) and server and client certificates.
    # This setting is required for [Cluster Peering](https://developer.hashicorp.com/consul/docs/connect/cluster-peering/k8s).
    enabled: true
  acls:

    # If true, the Helm chart will automatically manage ACL tokens and policies
    # for all Consul and consul-k8s-control-plane components.
    # This requires Consul >= 1.4.
    manageSystemACLs: true
  # The name (and tag) of the consul-dataplane Docker image used for the
  # connect-injected sidecar proxies and mesh, terminating, and ingress gateways.
  # @default: hashicorp/consul-dataplane:<latest supported version>
  imageConsulDataplane: "harbor.dockerregistry.com/gitops/hashicorp/consul-dataplane:1.1.0"


server:
  # The number of server agents to run. This determines the fault tolerance of
  # the cluster. Please refer to the [deployment table](https://developer.hashicorp.com/consul/docs/architecture/consensus#deployment-table)
  # for more information.
  replicas: 3
  # The StorageClass to use for the servers' StatefulSet storage. It must be
  # able to be dynamically provisioned if you want the storage
  # to be automatically created. For example, to use
  # local(https://kubernetes.io/docs/concepts/storage/storage-classes/#local)
  # storage classes, the PersistentVolumeClaims would need to be manually created.
  # A `null` value will use the Kubernetes cluster's default StorageClass. If a default
  # StorageClass does not exist, you will need to create one.
  # Refer to the [Read/Write Tuning](https://developer.hashicorp.com/consul/docs/install/performance#read-write-tuning)
  # section of the Server Performance Requirements documentation for considerations
  # around choosing a performant storage class.
  #
  # ~> **Note:** The [Reference Architecture](https://developer.hashicorp.com/consul/tutorials/production-deploy/reference-architecture#hardware-sizing-for-consul-servers)
  # contains best practices and recommendations for selecting suitable
  # hardware sizes for your Consul servers.
  # @type: string
  storageClass: local

# Values that configure running a Consul client on Kubernetes nodes.
client:
  extraConfig: |
    {"advertise_reconnect_timeout": "15m"}

# Values that configure the Consul UI.
ui:
  # If true, the UI will be enabled. This will
  # only _enable_ the UI, it doesn't automatically register any service for external
  # access. The UI will only be enabled on server agents. If `server.enabled` is
  # false, then this setting has no effect. To expose the UI in some way, you must
  # configure `ui.service`.
  # @default: global.enabled
  # @type: boolean
  enabled: true
  # Configure the service for the Consul UI.
  service:
    # The service type to register.
    # @type: string
    type: LoadBalancer

# Configures the automatic Connect sidecar injector.
connectInject:
  # True if you want to enable connect injection. Set to "-" to inherit from
  # global.enabled.
  enabled: false
```

```shell
helm install consul hashicorp/consul --values config.yaml --create-namespace --namespace consul --version "1.0.6"

# 离线部署
helm pull hashicorp/consul --version "1.0.6"
tar xf consul-1.0.6.tgz

vim consul/values.yaml ## 修改配置，具体也是修改上面config.yaml 的各处配置，另外需要将镜像地址改为本地的，如：
  image: "harbor.dockerregistry.com/gitops/hashicorp/consul:1.14.5"
  imageK8S: harbor.dockerregistry.com/gitops/hashicorp/consul-k8s-control-plane:1.0.6
  imageConsulDataplane: "harbor.dockerregistry.com/gitops/hashicorp/consul-dataplane:1.1.0"

helm install consul ./consul --create-namespace --namespace consul

# 参考这个配置
global:
  name: consul
  image: "harbor.dockerregistry.com/gitops/hashicorp/consul:1.13.1"
  imageK8S: harbor.dockerregistry.com/gitops/hashicorp/consul-k8s-control-plane:0.47.1
  gossipEncryption:
    autoGenerate: true
  tls:
    enabled: true
  acls:
    manageSystemACLs: true
server:
  replicas: 3
  storageClass: local
client:
  extraConfig: |
    {"advertise_reconnect_timeout": "15m"}
ui:
  enabled: true
connectInject:
  enabled: false

# 拷贝原配置后修改
cp consul/values.yaml .
vim consul/values.yaml

# 离线部署
helm install consul ./consul --values values.yaml  --create-namespace --namespace consul

# 离线更新配置
helm upgrade consul ./consul --values values.yaml -n consul

# ！！ 需要注意，前面consul 开启了全局全局tls ，在服务暴露后需要使用https 访问才行

# 验证
kubectl exec --stdin --tty consul-server-0 --namespace consul -- /bin/sh
$ consul members
$ exit

```



## 3 物理部署

```shell
### 安装
#  三台机器均执行
yum install -y yum-utils
yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
yum -y install consul
consul -v
mkdir -p /data/consul-data


# node1 执行
nohup consul agent -server -ui  -bootstrap-expect=1 -node=node1 -bind 10.0.1.1 -client=0.0.0.0 -data-dir=/data/consul-data/    > /data/consul-data/consul.log 2>&1 & 
tail -1000f /data/consul-data/consul.log

# node2 执行
nohup consul agent -node=node2 -bind 10.0.1.1 -client=0.0.0.0 -data-dir=/data/consul-data/ -join=10.0.1.2 > /data/consul-data/consul.log 2>&1 &
tail -1000f /data/consul-data/consul.log

# node4 执行
nohup consul agent -node=node4 -bind 10.0.1.1 -client=0.0.0.0 -data-dir=/data/consul-data/ -join=10.0.1.4 > /data/consul-data/consul.log 2>&1 &
tail -1000f /data/consul-data/consul.log


### 卸载
# node 节点，杀死进程
ps -ef | grep consul
kill -9 PID

# leader
consul force-leave  -prune  node4
consul force-leave  -prune  node2
```