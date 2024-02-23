## 1 Prometheus-online

### 1.1 Prometheus Operator

[Prometheus Operator 官方文档](https://prometheus-operator.dev/docs/prologue/quick-start/
https://github.com/kubesphere/kubekey)

### 1.1.1 准备k8s环境

```shell
## 准备k8s v1.23.10 环境

## 1. docker环境初始化
## 上传附件init-docker 目录脚本，修改自定义内容后执行
./init-docker



## 2. kubekey工具安装
mkdir -p kube
export KKZONE=cn
# curl -sfL https://get-kk.kubesphere.io | sh -
# https://github.com/kubesphere/kubekey/releases/download/v3.0.7/kubekey-v3.0.7-linux-amd64.tar.gz
# 上传kk附件
tar xf kubekey-v3.0.7-linux-amd64.tar.gz -C /usr/bin/
chmod +x /usr/bin/kk
kk version


## 3. k8s环境准备
# kk create config
echo -e "export KKZONE=cn\n" >>/etc/profile
source  /etc/profile

kk create cluster --with-kubernetes v1.25.3 --container-manager containerd

kubectl get pod -A

## 4. 自动补全
sudo yum -y install bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)


## 5 containerd 配置更新，并重启服务
$ vim /etc/containerd/config.toml
...
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]   ## 新增
          endpoint = ["https://registry.k8s.io"]  ## 新增
          
$ systemctl restart containerd

## 6 导入镜像，这里按实际需要导入即可，注意导入到指定命名空间下，否则找不到
ctr -n k8s.io i import adap.tar
```

#### 1.1.2 部署kube-prometheus

```shell
## 1. kube-prometheus部署文件准备
# git clone https://github.com/prometheus-operator/kube-prometheus.git
## 上传kube-prometheus部署文件

## 2. kube-prometheus部署
# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources 
mkdir -p monitoring
## 上传附件到上面目录
cd monitoring/kube-prometheus
kubectl apply --server-side -f manifests/setup
# Wait until the "servicemonitors" CRD is created. The message "No resources found" means success in this context. 
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done 
kubectl create -f manifests/
```

#### 1.1.3 访问kube-prometheus

```shell
## https://k8slens.dev/
## 借助Lens 工具，做服务端口暴露后查看，过程略
```

#### 1.1.4 移除kube-prometheus

```shell
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```

### 1.2 安装Operator

#### 1.2.1 部署Operator

```shell
# 以安装 CRD 并将 Operator 部署到`default`命名空间中
# yum -y install jq
# LATEST=$(curl -s https://api.github.com/repos/prometheus-operator/prometheus-operator/releases/latest | jq -cr .tag_name) 
# curl -sL https://github.com/prometheus-operator/prometheus-operator/releases/download/${LATEST}/bundle.yaml | kubectl create -f -

cd monitoring/
mkdir -p operator

# 上面的方式可能无法正常执行，需要离线下载后上传附件：bundle.yaml
# https://github.com/prometheus-operator/prometheus-operator/releases/download/v0.65.1/bundle.yaml


cd monitoring/operator
kubectl apply --server-side -f bundle.yaml --force-conflicts  ## --server-side  解决官方代码BUG

# 可以使用以下命令检查是否完成
kubectl wait --for=condition=Ready pods -l app.kubernetes.io/name=prometheus-operator -n default
```

#### 1.2.2 部署demo

```shell
# 创建工作负载
mkdir -p demo
cat > deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: fabxc/instrumented_app
        ports:
        - name: web
          containerPort: 8080
EOF

# 创建服务
cat > svc.yaml<<EOF
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
EOF

# 创建监控发现
cat > demo/service-monitor.yaml <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
EOF

# 执行整个目录
kubectl apply -f demo
```

#### 1.2.3 部署服务账户等

```shell
mkdir -p sa
vim sa/sa.yaml
vim sa/cr.yaml
vim sa/crb.yaml
vim sa/prome.yaml

kubectl apply -f sa
kubectl get -n default prometheus prometheus -w
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
```

#### 1.2.4 启用管理 API

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
      # 将这里的false 改为true
  enableAdminAPI: true
```





## 3 Prometheus-offline



### 2.1 Install Kube-prometheus

> ⚠️ Warning:!! 注意兼容性，[官方文档](https://github.com/prometheus-operator/kube-prometheus#compatibility)

```shell
# https://prometheus-operator.dev/docs/prologue/quick-start/#deploy-kube-prometheus
mkdir -p kube-prometheus  # upload files.
cd kube-prometheus/

# uploaded files.
root@master1:~/kube-prometheus# ll 
total 4
drwxr-xr-x 3 root root 4096 Jan 12 10:09 manifests


kubectl get crd | grep 'monitoring.coreos.com' # 查询crd，确认并无旧环境再部署

# 安装crd
# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources
kubectl create -f manifests/setup

# Wait until the "servicemonitors" CRD is created. The message "No resources found" means success in this context.
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done  # like : "No resources found"

# 查看crd
root@master1:~/kube-prometheus# kubectl get crd | grep 'monitoring.coreos.com'
alertmanagerconfigs.monitoring.coreos.com             2024-01-12T02:23:09Z
alertmanagers.monitoring.coreos.com                   2024-01-12T02:23:10Z
podmonitors.monitoring.coreos.com                     2024-01-12T02:23:10Z
probes.monitoring.coreos.com                          2024-01-12T02:23:10Z
prometheusagents.monitoring.coreos.com                2024-01-12T02:23:11Z
prometheuses.monitoring.coreos.com                    2024-01-12T02:23:10Z
prometheusrules.monitoring.coreos.com                 2024-01-12T02:23:11Z
scrapeconfigs.monitoring.coreos.com                   2024-01-12T02:23:11Z
servicemonitors.monitoring.coreos.com                 2024-01-12T02:23:11Z
thanosrulers.monitoring.coreos.com                    2024-01-12T02:23:11Z

# 部署kube-prometheus
kubectl create -f manifests/


# 卸载kube-prometheus
#kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```



官方文档：https://prometheus-operator.dev/docs/operator/storage/

> 因为默认是`kube-prometheus ` 使用的存储是未做持久化的，所以这里需要走本地`NFS `来做持久化。



```shell
root@master1:~/kube-prometheus# kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup ## 卸载清理环境

root@master1:~/kube-prometheus# cp  manifests/prometheus-prometheus.yaml{,.bak}  # 备份原配置
root@master1:~/kube-prometheus# ll manifests/prometheus-prometheus.yaml*
-rw-r--r-- 1 root root 1301 Jan 12 10:06 manifests/prometheus-prometheus.yaml
-rw-r--r-- 1 root root 1301 Jan 15 15:45 manifests/prometheus-prometheus.yaml.bak
root@master1:~/kube-prometheus# vim manifests/prometheus-prometheus.yaml # 末尾粘贴自己的PVC内容
root@master1:~/kube-prometheus# tail manifests/prometheus-prometheus.yaml
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: default
        resources:
          requests:
            storage: 50Gi
            
            
kubectl create -f manifests/setup  # 再次部署
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done  # it's ready like : "No resources found"
kubectl get crd | grep 'monitoring.coreos.com'
kubectl create -f manifests/


# 查看pvc 
root@master1:~/kube-prometheus# kubectl get pvc -n monitoring 
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-80cb9be7-e70a-4241-8375-efa5bcf6cd0b   50Gi       RWO            default        62s
prometheus-k8s-db-prometheus-k8s-1   Bound    pvc-5d17e9a8-f7e9-4916-9b6f-4302306656c3   50Gi       RWO            default        61s
```



> ⚠️ Warning:!! 请务必确保所有节点安装`NFS 客户端`, 否则PVC 可能无法正常创建成功或者PV 无法正常挂载

```shell
apt-get install nfs-common -y
```





### 2.2 Create ingress

1. create `ingress` for prometheus

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/auth-realm: Authentication Required
    nginx.ingress.kubernetes.io/auth-secret: prometheus-basic-auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/proxy-body-size: 2048m
    nginx.ingress.kubernetes.io/proxy-connect-timeout: '1800'
    nginx.ingress.kubernetes.io/proxy-read-timeout: '1800'
    nginx.ingress.kubernetes.io/proxy-send-timeout: '1800'
    nginx.org/client-max-body-size: 2048m
    nginx.org/keepalive: 75s
    nginx.org/proxy-buffering: 'false'
    nginx.org/proxy-connect-timeout: 1800s
    nginx.org/proxy-read-timeout: 1800s
    nginx.org/proxy-send-timeout: 1800s
spec:
  ingressClassName: nginx
  rules:
    - host: prometheus.k8scluster.com
      http:
        paths:
          - path: /prometheus(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: prometheus-k8s
                port:
                  name: web
    - host: alertmanager.k8scluster.com
      http:
        paths:
          - path: /alertmanager(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: alertmanager-main
                port:
                  name: web
```



2. create `http auth`

```shell
htpasswd -c auth admin # 输入密码
```



3. create `secret`

```shell
kubectl create secret generic prometheus-basic-auth --from-file=auth -n monitoring
```



## 4 Uninstall kubesphere prometheus


> 参考官方文档： https://www.kubesphere.io/zh/docs/v3.4/faq/observability/byop/



```shell
kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/alertmanager/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/devops/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/etcd/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/grafana/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/kube-state-metrics/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/node-exporter/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/upgrade/ 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/prometheus-rules-v1.16\+.yaml 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/prometheus-rules.yaml 2>/dev/null

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/prometheus 2>/dev/null

# Uncomment this line if you don't have Prometheus managed by Prometheus Operator in other namespaces.

kubectl -n kubesphere-system exec $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}') -- kubectl delete -f /kubesphere/kubesphere/prometheus/init/ 2>/dev/null

```





```shell
kubectl -n kubesphere-monitoring-system delete pvc `kubectl -n kubesphere-monitoring-system get pvc | grep -v VOLUME | awk '{print $1}' |  tr '\n' ' '`

```





> 默认的命令无法清理彻底，这里是补充命令：



```shell
kubectl -n kubesphere-monitoring-system delete deployment notification-manager-deployment notification-manager-operator prometheus-operator  ## 多执行两遍


kubectl -n kubesphere-monitoring-system delete svc notification-manager-controller-metrics notification-manager-svc notification-manager-webhook prometheus-operator


kubectl get all -n kubesphere-monitoring-system


kubectl get crd | grep 'monitoring.coreos.com' # 查询crd

# 删除crd
kubectl delete crd prometheusagents.monitoring.coreos.com
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
#kubectl delete crd scrapeconfigs.monitoring.coreos.com  # 这个可能并不存在


# 删除clusterrole 、 clusterrolebinding
kubectl delete clusterrolebinding kubesphere-prometheus-operator
kubectl delete clusterrole kubesphere-prometheus-operator

```

