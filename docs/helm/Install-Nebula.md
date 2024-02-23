
## 1 `operator` 部署

[`Github Chart` 仓库地址](https://github.com/vesoft-inc/nebula-operator)

[官方文档](https://docs.nebula-graph.com.cn/3.4.1/nebula-operator/2.deploy-nebula-operator/)

```shell
# 添加仓库并查找
helm repo add nebula-operator https://vesoft-inc.github.io/nebula-operator/charts
helm repo update
helm search repo -l nebula-operator

# 拉取chart包到本地
helm pull nebula-operator/nebula-operator --version=1.4.2
tar xf nebula-operator-1.4.2.tgz 

# 安装
# 映射环境信息
export NEBULA_CLUSTER_NAME=nebula         # NebulaGraph 集群的名字。
export NEBULA_CLUSTER_NAMESPACE=nebula    # NebulaGraph 集群所处的命名空间的名字。
export STORAGE_CLASS_NAME=nfs             # NebulaGraph 集群的 StorageClass。
# 写入系统环境变量，过程略

# 创建命名空间
kubectl create namespace "${NEBULA_CLUSTER_NAMESPACE}"

# helm 部署operator
helm install "${NEBULA_CLUSTER_NAME}"-"operator" ./nebula-operator \
    --namespace="${NEBULA_CLUSTER_NAMESPACE}" \
    --set nameOverride="${NEBULA_CLUSTER_NAME}" \
    --set nebula.storageClassName="${STORAGE_CLASS_NAME}" \
    --set nebula.storaged.replicas=3

# helm 卸载operator
helm uninstall "${NEBULA_CLUSTER_NAME}" --namespace="${NEBULA_CLUSTER_NAMESPACE}"

# 清理crd
kubectl delete crd nebulaclusters.apps.nebula-graph.io
```


## 2 `nebula` 集群部署

```shell
# 添加仓库并查找
helm repo add nebula-operator https://vesoft-inc.github.io/nebula-operator/charts
helm repo update
helm search repo -l nebula-cluster

# 拉取chart包到本地
helm pull nebula-operator/nebula-cluster --version=1.4.2
tar xf nebula-cluster-1.4.2.tgz 

# helm 部署集群
helm install "${NEBULA_CLUSTER_NAME}"-"cluster" ./nebula-cluster \
    --namespace="${NEBULA_CLUSTER_NAMESPACE}" \
    --set nameOverride=${NEBULA_CLUSTER_NAME} \
    --set nebula.storageClassName="${STORAGE_CLASS_NAME}"

# helm 卸载集群
helm uninstall "${NEBULA_CLUSTER_NAME}"-"cluster" --namespace="${NEBULA_CLUSTER_NAMESPACE}"

# 查看pod
kubectl -n "${NEBULA_CLUSTER_NAMESPACE}" get pod -l "app.kubernetes.io/cluster=${NEBULA_CLUSTER_NAME}"

# 查看svc
kubectl get service -n "${NEBULA_CLUSTER_NAMESPACE}"


# 修改配置
kubectl edit nebulaclusters.apps.nebula-graph.io ${NEBULA_CLUSTER_NAMESPACE} -n "${NEBULA_CLUSTER_NAMESPACE}"

# 添加enable_authorize和auth_type，Graph 服务对应的 ConfigMap（nebula-graphd）中的配置将被覆盖
...
    config: //为 Graph 服务自定义参数。
      "enable_authorize": "true"
      "auth_type": "password"
# 修改pv自动回收，定义在删除集群后自动删除 PVC 以释放数据
...
spec:
  enablePVReclaim: true  //设置其值为 true
# 添加enableAutoBalance并设置其值为true，扩容后自动均衡 Storage 数据
...
  schedulerName: default-scheduler
  storaged:
    enableAutoBalance: true   //将其值设置为 true 时表示扩容后自动均衡 Storage 数据
# 修改集群的滚动更新策略
...
  storaged:
    enableForceUpdate: false // 设置为 false 时，表示迁移分片 Leader 副本，以保证集群的读写可用性，默认值为false。

```


## 3 `Studio` 图形界面部署

[官方文档-Helm部署`Studio`](https://docs.nebula-graph.com.cn/3.4.1/nebula-studio/deploy-connect/st-ug-deploy/#helm_studio)

[官方文档](https://docs.nebula-graph.com.cn/3.4.1/nebula-studio/about-studio/st-ug-what-is-graph-studio/)


>`NebulaGraph Studio（简称 Studio）`是一款可以通过 `Web` 访问的开源图数据库可视化工具，搭配 `NebulaGraph` 内核使用，提供构图、数据导入、编写 `nGQL` 查询等一站式服务。用户可以在 [`NebulaGraph GitHub`](https://github.com/vesoft-inc/nebula-studio) 仓库中查看最新源码。

```shell
# 安装
helm install "${NEBULA_CLUSTER_NAME}"-"studio" ./nebula-studio \
    --namespace="${NEBULA_CLUSTER_NAMESPACE}" \
    --set nameOverride=${NEBULA_CLUSTER_NAME} \
    --set nebula.storageClassName="${STORAGE_CLASS_NAME}" \
    --set service.type="NodePort" 

# 卸载
helm uninstall "${NEBULA_CLUSTER_NAME}"-"studio" --namespace="${NEBULA_CLUSTER_NAMESPACE}"

```

## 4 客户端连接

>`NebulaGraph Console`：原生 `CLI` 客户端
>`Nebula Console` 是 `NebulaGraph` 的原生命令行客户端，用于连接 `NebulaGraph` 集群并执行查询，同时支持管理参数、导出命令的执行结果、导入测试数据集等功能。

[`GitHub 发布页`](https://github.com/vesoft-inc/nebula-console/releases)
[官方文档](https://docs.nebula-graph.com.cn/3.4.1/nebula-console/)

```shell
# 客户端连接
# 服务端已启用身份认证，默认root/nebula
/usr/bin/nebula-console -addr 10.0.1.12 -port 30741 -u root -p nebula

Welcome to Nebula Graph!

(root@nebula) [(none)]>
```
