
[Velero官方文档](https://velero.io/docs/v1.10/customize-installation/)

## 1  安装客户端

[客户端下载地址](https://github.com/vmware-tanzu/velero/releases)

```shell
# 下载
wget https://github.com/vmware-tanzu/velero/releases/download/v1.10.2/velero-v1.10.2-linux-amd64.tar.gz

# 安装客户端
tar xf velero-v1.10.2-linux-amd64.tar.gz
cp velero-v1.10.2-linux-amd64/velero /usr/bin/

# 查看版本
velero version
```


## 2 安装服务端

>这里选用本地`Minio` 作为备份存储


### 2.1 插件兼容性


[`s3` 插件与`velero` 版本兼容性的官方说明](https://github.com/vmware-tanzu/velero-plugin-for-aws#compatibility)


| Plugin Version | Velero Version |
|----------------|----------------|
| v1.6.x         | v1.10.x        |
| v1.5.x         | v1.9.x         |
| v1.4.x         | v1.8.x         |
| v1.3.x         | v1.7.x         |
| v1.2.x         | v1.6.x         |
| v1.1.x         | v1.5.x         |
| v1.1.x         | v1.4.x         |
| v1.0.x         | v1.3.x         |
| v1.0.x         | v1.2.0         |


### 2.2 部署服务端

[官方操作说明](https://velero.io/docs/v1.10/contributions/minio/)


```shell

# 创建minio 桶，过程略
fish-velero


# 创建minio 认证文件
$ cd velero-v1.10.2-linux-amd64

"
cat <<EOF | sudo tee credentials-velero
[default]
aws_access_key_id = "admin"
aws_secret_access_key = "admin@123"
EOF
"

# 使用客户端安装服务端
# 前提： 当前节点能有集群认证权限，可以通过kubectl 进行部署
# 自定义minio 桶名：fish-velero
# 本地minio 服务器地址: http://10.3.1.250:9000
# plugins 对应插件镜像名，这里用的是本地镜像
$ velero install \
    --provider aws \
    --plugins harbor.dockerregistry.com/gitops/velero/velero-plugin-for-aws:v1.6.0 \
    --bucket fish-velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://10.1.1.1:9000


# 部署示例应用
$ kubectl apply -f examples/nginx-app/base.yaml

# 检查是否已成功创建 Velero 和 nginx 部署
$ kubectl get deployments -l component=velero --namespace=velero
$ kubectl get deployments --namespace=nginx-example

```


### 2.3 创建备份

[官方文档](https://velero.io/docs/v1.10/contributions/minio/)

```shell
# 为与标签选择器匹配的任何对象创建备份app=nginx
$ velero backup create nginx-backup --selector app=nginx


# 删除资源
kubectl delete namespace nginx-example
# 检查资源
kubectl get deployments --namespace=nginx-example
kubectl get services --namespace=nginx-example
kubectl get namespace/nginx-example
```

### 2.4 查看备份

```shell
$ velero backup get
```

### 2.5 备份恢复

[官方文档](https://velero.io/docs/v1.10/contributions/minio/)

```shell
# 使用备份进行恢复
$ velero restore create --from-backup nginx-backup

# 查看恢复记录
$ velero restore get
# 查看恢复详细信息
$ velero restore describe $RESTORE_NAME
```

### 2.6 清理备份

```shell
#Examples:
  # Delete a backup named "backup-1".
  velero backup delete backup-1

  # Delete a backup named "backup-1" without prompting for confirmation.
  velero backup delete backup-1 --confirm

  # Delete backups named "backup-1" and "backup-2".
  velero backup delete backup-1 backup-2

  # Delete all backups triggered by schedule "schedule-1".
  velero backup delete --selector velero.io/schedule-name=schedule-1

  # Delete all backups.
  velero backup delete --all

```

[官方文档](https://velero.io/docs/v1.10/contributions/minio/)

```shell
# 清理备份
velero backup delete $BACKUP_NAME

# 查看备份
velero backup get $BACKUP_NAME
```

### 2.7 完全卸载

[官方文档](https://velero.io/docs/v1.10/contributions/minio/)

```shell
# 清理velero 服务端
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero

# 清理示例应用
kubectl delete -f examples/nginx-app/base.yaml
```



## 3 备份原理

[官方文档](https://velero.io/docs/v1.10/how-velero-works/)

### 3.1 对象存储同步

>如果存储桶中有格式正确的备份文件，但 `Kubernetes API` 中没有对应的备份资源，`Velero` 会将对象存储的信息同步到 `Kubernetes`。这允许还原功能在集群迁移场景中工作，其中原始备份对象不存在于新集群中。

>如果`Completed`备份对象存在于 `Kubernetes` 但不在对象存储中，它将从 `Kubernetes` 中删除，因为备份 `tarball` 不再存在。 `Failed`或`PartiallyFailed`备份不会被对象存储同步删除。


### 3.2 设置备份过期

>创建备份时，可以通过添加标志来指定 `TTL`（生存时间）`--ttl <DURATION>`。如果 `Velero` 发现现有备份资源已过期，它会删除:

- 备份资源
- 来自云对象存储的备份文件
- 所有 `PersistentVolume` 快照
- 所有关联的恢复

>`TTL` 标志允许用户以小时、分钟和秒的形式指定备份保留期`--ttl 24h0m0s`。如果未指定，将应用默认的 `TTL` 值 `30` 天。

>如果备份删除失败，`velero.io/gc-failure=<Reason>`则会在备份自定义资源上添加标签。
>您可以使用此标签来筛选和选择未能删除的备份。失败可能的原因是：

- `BSLNotFound`：找不到备份存储位置
- `BSLCannotGet`：由于未找到以外的原因，无法从 API 服务器检索备份存储位置
- `BSLReadOnly`：备份存储位置是只读的


### 3.3 集群`API`版本

>`Velero` 使用 `the Kubernetes API server’s preferred version ` 为每个组/资源备份资源。
>`When restoring a resource, this same API group/version must exist in the target cluster in order for the restore to be successful.` 
>简言之，当初建立备份的 `k8s` 集群版本与要还原到的 `k8s`集群版本需要尽可能一致，否则会因为 `API`版本差异导致恢复异常。

### 3.4 恢复工作流程

>When you run `velero restore create`  :

- `The Velero client makes a call to the Kubernetes API server to create a Restore object.`
	客户端会请求`k8s api` 去创建恢复的资源对象

- `The RestoreController notices the new Restore object and performs validation.`
	恢复控制器会通知备份服务端将需要创建一个恢复对象，然后准备开始执行验证

- `The RestoreController fetches the backup information from the object storage service. It then runs some preprocessing on the backed up resources to make sure the resources will work on the new cluster. For example, using the backed-up API versions to verify that the restore resource will work on the target cluster`.	
	恢复控制器会获取备份资源对象，进行预处理来确保要恢复的资源将可以在新的恢复位置上可以进行正常恢复。

- `The RestoreController starts the restore process, restoring each eligible resource one at a time.`
	恢复控制器会执行恢复工作，一次性执行可以用来恢复的合格资源进行恢复。


### 3.5 备份工作流程


>When you run `velero backup create test-backup` :

1. `The Velero client makes a call to the Kubernetes API server to create a Backup object.`

2. `The BackupController notices the new Backup object and performs validation.`

3. `The BackupController begins the backup process. It collects the data to back up by querying the API server for resources.`

4. `The BackupController makes a call to the object storage service – for example, AWS S3 – to upload the backup file.`

`By default, velero backup create makes disk snapshots of any persistent volumes. You can adjust the snapshots by specifying additional flags. Run velero backup create --help to see available flags. Snapshots can be disabled with the option --snapshot-volumes=false.`

![[Pasted image 20230330144306.png]]


## 4 备份方式

[官方文档](https://velero.io/docs/v1.10/how-velero-works/)

### 4.1 按需备份

#### 4.1.1 指定特定种类资源的备份顺序

>要以特定顺序备份特定 `Kind` 的资源，请使用选项 ` --ordered-resources` 指定映射 `Kinds` 到该 `Kind` 特定资源的有序列表。资源名称以逗号分隔，其名称格式为“名称空间/资源名称”。对于集群范围资源，只需使用资源名称。映射中的键值对以分号分隔。种类名称是复数形式。

```shell
# --include-cluster-resources=true  默认就是备份整个集群资源
# --ordered-resources 备份顺序列表清单
# --include-namespaces 指定命名空间
# pod/sts 是命名空间级别资源
# pv 是集群级资源
velero backup create backupName --include-cluster-resources=true --ordered-resources 'pods=ns1/pod1,ns1/pod2;persistentvolumes=pv4,pv8' --include-namespaces=ns1
velero backup create backupName --ordered-resources 'statefulsets=ns1/sts1,ns1/sts0' --include-namespaces=ns1
```

#### 4.1.2 从备份中排除特定项目

```shell
# velero.io/exclude-from-backup=true 就是给资源打的标签
kubectl label -n <ITEM_NAMESPACE> <RESOURCE>/<NAME> velero.io/exclude-from-backup=true
```



### 4.2 计划备份

>时间间隔由 [`Cron` 表达式](https://en.wikipedia.org/wiki/Cron) 指定，`Velero` 使用名称保存根据计划创建的备份`<SCHEDULE NAME>-<TIMESTAMP>`，其中`<TIMESTAMP>`格式为`YYYYMMDDhhmmss`

>`Cron` 计划使用以下格式：

```shell
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
# │ │ │ │ │                                   7 is also Sunday on some systems)
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

>语法格式：
```shell
# 创建备份计划
velero schedule create ${BACKUP-SCHEDULE-NAME} --schedule="* * * * *" [flags]

# 清理备份计划
velero schedule delete ${BACKUP-SCHEDULE-NAME}

# 立即触发备份计划
velero backup create --from-schedule ${BACKUP-SCHEDULE-NAME}
```

>示例1，每天凌晨3点开始备份：

```shell
# 备份计划中指定备份周期，备份对象
velero schedule create example-schedule --schedule="0 3 * * *"  --selector app=nginx --include-namespaces nginx-example
# 备份计划中指定命名空间，默认备份所有命名空间
velero schedule create example-schedule2 --schedule="0 3 * * *" --include-namespaces nginx-example 
# 备份计划中，设置备份过期时间
velero schedule create example-schedule3 --schedule="40 9 * * *" --include-namespaces="nginx-example" --ttl="72h"

# 立即触发
velero backup create --from-schedule example-schedule3

# 查看备份
$ velero backup get
NAME                               STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
example-schedule3-20230330083511   Completed   0        0          2023-03-30 16:35:11 +0800 CST   2d        default            <none>
nginx-backup                       Completed   0        0          2023-03-30 10:29:56 +0800 CST   29d       default            app=nginx
```


## 5 查看日志

```shell
# 查看deploy 日志
$ kubectl logs -f --tail 1000  deployment/velero -n velero


# 检查备份存储位置的状态，显示备份存储位置的当前状态以及可能存在的任何错误
$ velero backup-location get -n velero

```

## 6 调优


### 6.1 修改日志级别

>默认`info`

```yaml
...
      containers:
        - name: velero
          image: 'harbor.dockerregistry.com/gitops/velero/velero:v1.10.2'
          command:
            - /velero
          args:
            - server
            - '--features='
            - '--uploader-type=restic'
            - '--log-level=debug'  ## 加在这里
```


### 6.2 命令自动补全

>先实现 `kubectl` 命令自动补全

```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

>再实现`velero` 命令自动补全

```shell
source /usr/share/bash-completion/bash_completion
echo 'source <(velero completion bash)' >>~/.bashrc
velero completion bash >/etc/bash_completion.d/velero
echo 'alias v=velero' >>~/.bashrc
echo 'complete -F __start_velero v' >>~/.bashrc
source ~/.bashrc
```


### 6.3 时区问题调整

>默认使用的是`UTC` 时间

```yml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: velero
  namespace: velero
  labels:
    component: velero
  annotations:
    deployment.kubernetes.io/revision: '6'
spec:
  replicas: 1
  selector:
    matchLabels:
      deploy: velero
  template:
    metadata:
      creationTimestamp: null
      labels:
        component: velero
        deploy: velero
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: '8085'
        prometheus.io/scrape: 'true'
    spec:
      volumes:
        - name: plugins
          emptyDir: {}
        - name: scratch
          emptyDir: {}
        - name: cloud-credentials
          secret:
            secretName: cloud-credentials
            defaultMode: 420
        - name: host-time             ######################### 这里，同步主机时间
          hostPath:
            path: /etc/localtime
            type: ''
      initContainers:
        - name: gitops-velero-velero-plugin-for-aws
          image: 'harbor.dockerregistry.com/gitops/velero/velero-plugin-for-aws:v1.6.0'
          resources: {}
          volumeMounts:
            - name: plugins
              mountPath: /target
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      containers:
        - name: velero
          image: 'harbor.dockerregistry.com/gitops/velero/velero:v1.10.2'
          command:
            - /velero
          args:
            - server
            - '--features='
            - '--uploader-type=restic'
            - '--log-level=debug'
          ports:
            - name: metrics
              containerPort: 8085
              protocol: TCP
          env:
            - name: VELERO_SCRATCH_DIR
              value: /scratch
            - name: VELERO_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: LD_LIBRARY_PATH
              value: /plugins
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: /credentials/cloud
            - name: AWS_SHARED_CREDENTIALS_FILE
              value: /credentials/cloud
            - name: AZURE_CREDENTIALS_FILE
              value: /credentials/cloud
            - name: ALIBABA_CLOUD_CREDENTIALS_FILE
              value: /credentials/cloud
          resources:
            limits:
              cpu: '1'
              memory: 512Mi
            requests:
              cpu: 500m
              memory: 128Mi
          volumeMounts:
            - name: plugins
              mountPath: /plugins
            - name: scratch
              mountPath: /scratch
            - name: cloud-credentials
              mountPath: /credentials
            - name: host-time               ######################### 这里，同步主机时间
              readOnly: true
              mountPath: /etc/localtime
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      serviceAccountName: velero
      serviceAccount: velero
      securityContext: {}
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```


## 7 安装选项

[官方文档](https://velero.io/docs/main/customize-installation/#plugins

```shell
# minio 做外部备份存储的备份服务端安装方式
$ velero install \
    --provider aws \
    --plugins harbor.dockerregistry.com/gitops/velero/velero-plugin-for-aws:v1.6.0 \
    --bucket fish-velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://10.1.1.1:9000
```

### 7.1 指定命名空间

>默认服务端和客户端都是找的命名空间 `velero`

```shell
## 部署服务端时指定
# velero install [flags]
velero install \
--namespace xxx

## 客户端需要配置为一致
# velero client config set KEY=VALUE [KEY=VALUE]... [flags]
velero client config set namespace=$NAMESPACE_VALUE
```

## 8 备份选项

### 8.1 启用文件系统备份

```shell
## 部署服务端时指定
# velero install [flags]
velero install \
--use-node-agent
```


### 8.2 开源备份工具局限性

>综合评估，建议在使用开源备份工具 `restic` 、 `kopia` 时，使用独立或更高配置的服务器做为客户端，建议至少可用配置在`4C、4G` ，方便客户端再执行备份工作时有的资源可用，同时，`kopia` 相较来说更满足大部分场景需求。这里是[官方测评报告](https://velero.io/docs/main/performance-guidance/)

