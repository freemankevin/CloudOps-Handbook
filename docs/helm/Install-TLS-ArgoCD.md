## 1 Prerequisites

```bash
# need three worker nodes. 
```



## 2 Install Client

```bash
# 版本兼容性说明，请查看官方文档
# https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#tested-versions

# git clone -b v2.9.2 https://github.com/argoproj/argo-cd.git

mkdir -p argoCD && cd argoCD

# on maste1.
# upload files.

# view files.
root@master2:~/argoCD# ll
total 149856
drwxr-xr-x 24 root root      4096 Jan 16 20:57 argo-cd
-rw-r--r--  1 root root 153442642 Dec  7 17:31 argocd-linux-amd64
drwxr-xr-x  2 root root      4096 Jan 16 20:57 images


# mv file.
\cp -rvf argocd-linux-amd64 /usr/bin/argocd
chmod +x /usr/bin/argocd

# check it.
root@master1:~# argocd version 
argocd: v2.9.2+c5ea5c4
  BuildDate: 2023-11-20T17:37:53Z
  GitCommit: c5ea5c4df52943a6fff6c0be181fde5358970304
  GitTreeState: clean
  GoVersion: go1.21.4
  Compiler: gc
  Platform: linux/amd64
FATA[0000] Argo CD server address unspecified


# scp files.
scp -r  argocd-linux-amd64 master2:~/
scp -r  argocd-linux-amd64 master3:~/
scp -r  argocd-linux-amd64 worker1:~/
ssh master2 "\cp -rvf argocd-linux-amd64 /usr/bin/argocd && chmod +x /usr/bin/argocd" 
ssh master3 "\cp -rvf argocd-linux-amd64 /usr/bin/argocd && chmod +x /usr/bin/argocd" 
ssh worker1 "\cp -rvf argocd-linux-amd64 /usr/bin/argocd && chmod +x /usr/bin/argocd" 
```



## 3 Pull files.

```bash
# on worker1 
# upload files.

# view files.
root@worker1:~# ls -lh *.tar 
-rw-r--r-- 1 root root 414M Dec  7 17:46 argocd.tar
-rw-r--r-- 1 root root  93M Dec  7 17:46 dex.tar
-rw-r--r-- 1 root root  23M Dec  7 17:47 haproxy.tar
-rw-r--r-- 1 root root  30M Dec  7 17:46 redis.tar

# load images.
docker load -i xx.tar 
```



## 4 Install Server

```bash
# on master1 

# egrep -v '^#|^$' ha/namespace-install.yaml | grep -A 1 image
# docker pull ghcr.io/dexidp/dex:v2.37.0
# docker pull haproxy:2.6.14-alpine
# docker pull quay.io/argoproj/argocd:v2.9.2
# docker pull redis:7.0.11-alpine

# on worker nodes
docker load -i adapter.tar 
docker load -i argocd.tar 
docker load -i dex.tar 
docker load -i haproxy.tar

root@master2:~/argoCD# ll argo-cd/manifests/
total 2456
drwxr-xr-x  2 root root    4096 Jan 16 20:57 addons
drwxr-xr-x 11 root root    4096 Jan 16 20:57 base
drwxr-xr-x  2 root root    4096 Jan 16 20:57 cluster-install
drwxr-xr-x  4 root root    4096 Jan 16 20:57 cluster-rbac
drwxr-xr-x  2 root root    4096 Jan 16 20:57 core-install
-rw-r--r--  1 root root 1194391 Dec  7 17:26 core-install.yaml
drwxr-xr-x  2 root root    4096 Jan 16 20:57 crds
drwxr-xr-x  5 root root    4096 Jan 16 20:57 ha
-rw-r--r--  1 root root 1217616 Dec  7 17:26 install.yaml
drwxr-xr-x  2 root root    4096 Jan 16 20:57 namespace-install
-rw-r--r--  1 root root   59885 Dec  7 17:26 namespace-install.yaml
-rw-r--r--  1 root root    1775 Dec  7 17:26 README.md

cd argo-cd/manifests/
kubectl create namespace argocd
kubectl create -n argocd -f ha/install.yaml





root@master1:~/argocd/manifests# kubectl -n argocd get po 
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          3m8s
argocd-applicationset-controller-6f9d6cfd58-lg26b   1/1     Running   0          3m8s
argocd-dex-server-6df5d4f8c4-w7fkc                  1/1     Running   0          3m8s
argocd-notifications-controller-589b479947-wwz7r    1/1     Running   0          3m8s
argocd-redis-ha-haproxy-f66786cb6-nz7qg             1/1     Running   0          3m8s
argocd-redis-ha-haproxy-f66786cb6-snzwb             1/1     Running   0          3m8s
argocd-redis-ha-haproxy-f66786cb6-tjdcr             1/1     Running   0          3m8s
argocd-redis-ha-server-0                            3/3     Running   0          3m8s
argocd-redis-ha-server-1                            3/3     Running   0          111s
argocd-redis-ha-server-2                            1/3     Running   0          51s
argocd-repo-server-5b5df8f959-824rg                 1/1     Running   0          3m8s
argocd-repo-server-5b5df8f959-x4552                 1/1     Running   0          3m8s
argocd-server-5dcfd5ddd5-bgz4w                      1/1     Running   0          3m8s
argocd-server-5dcfd5ddd5-cz5tk                      1/1     Running   0          3m8s


root@master1:~/argocd/manifests# kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.105.46.153    <none>        7000/TCP,8080/TCP            3m31s
argocd-dex-server                         ClusterIP   10.99.148.154    <none>        5556/TCP,5557/TCP,5558/TCP   3m31s
argocd-metrics                            ClusterIP   10.98.209.89     <none>        8082/TCP                     3m31s
argocd-notifications-controller-metrics   ClusterIP   10.109.180.82    <none>        9001/TCP                     3m31s
argocd-redis-ha                           ClusterIP   None             <none>        6379/TCP,26379/TCP           3m30s
argocd-redis-ha-announce-0                ClusterIP   10.111.42.215    <none>        6379/TCP,26379/TCP           3m30s
argocd-redis-ha-announce-1                ClusterIP   10.98.233.90     <none>        6379/TCP,26379/TCP           3m30s
argocd-redis-ha-announce-2                ClusterIP   10.108.127.116   <none>        6379/TCP,26379/TCP           3m30s
argocd-redis-ha-haproxy                   ClusterIP   10.96.102.173    <none>        6379/TCP,9101/TCP            3m30s
argocd-repo-server                        ClusterIP   10.111.212.152   <none>        8081/TCP,8084/TCP            3m30s
argocd-server                             ClusterIP   10.102.152.4     <none>        80/TCP,443/TCP               3m30s
argocd-server-metrics                     ClusterIP   10.108.52.180    <none>        8083/TCP                     3m30s


```



## 5 Expose SVC

```bash
# 使用NodePort 方式暴露
# 80  对应 30884
# 443 对应 30885
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"nodePort": 30884, "port": 80}, {"nodePort": 30885, "port": 443}]}}'

root@master1:~/argocd/manifests# kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.105.46.153    <none>        7000/TCP,8080/TCP            5m18s
argocd-dex-server                         ClusterIP   10.99.148.154    <none>        5556/TCP,5557/TCP,5558/TCP   5m18s
argocd-metrics                            ClusterIP   10.98.209.89     <none>        8082/TCP                     5m18s
argocd-notifications-controller-metrics   ClusterIP   10.109.180.82    <none>        9001/TCP                     5m18s
argocd-redis-ha                           ClusterIP   None             <none>        6379/TCP,26379/TCP           5m17s
argocd-redis-ha-announce-0                ClusterIP   10.111.42.215    <none>        6379/TCP,26379/TCP           5m17s
argocd-redis-ha-announce-1                ClusterIP   10.98.233.90     <none>        6379/TCP,26379/TCP           5m17s
argocd-redis-ha-announce-2                ClusterIP   10.108.127.116   <none>        6379/TCP,26379/TCP           5m17s
argocd-redis-ha-haproxy                   ClusterIP   10.96.102.173    <none>        6379/TCP,9101/TCP            5m17s
argocd-repo-server                        ClusterIP   10.111.212.152   <none>        8081/TCP,8084/TCP            5m17s
argocd-server                             NodePort    10.102.152.4     <none>        80:30884/TCP,443:30885/TCP   5m17s
argocd-server-metrics                     ClusterIP   10.108.52.180    <none>        8083/TCP                     5m17s
```



## 6 Change default passwd

```bash
# 获取默认密码
root@master1:~/argocd/manifests# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
eBQwQdv60T9okZTl


# 或者用命令行工具获取
root@master1:~/argocd/manifests# argocd admin initial-password -n argocd
eBQwQdv60T9okZTl

 This password must be only used for first time login. We strongly recommend you update the password using `argocd account update-password`.

# 命令行工具登陆
root@master1:~/argocd/manifests# argocd login 192.168.1.120:30884
# WARNING: server certificate had error: tls: failed to verify certificate: x509: cannot validate certificate for 192.168.1.120 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? y
# Username: admin
# Password: 
# 'admin:login' logged in successfully
# Context '192.168.1.120:30884' updated


root@master1:~/argocd/manifests# argocd account update-password
# *** Enter password of currently logged in user (admin):     # <-- 写入前面获取的默认密码
# *** Enter new password for user admin:                      # <-- 写入要修改的新密码
# *** Confirm new password for user admin:                    # <-- 再次，写入要修改的新密码
# Password updated
# Context '192.168.1.120:30884' updated


# 获取集群上下文
root@master1:~/argocd/manifests# kubectl config get-contexts -o name
kubernetes-admin@kubernetes

# 获取集群列表
root@master1:~/argocd/manifests# argocd cluster list
SERVER                          NAME        VERSION  STATUS   MESSAGE                                                  PROJECT
https://kubernetes.default.svc  in-cluster           Unknown  Cluster has no applications and is not being monitored.  

# 退出登陆（若需要）
argocd logout 192.168.1.120:30884

```





## 7 Change rolebinding

```bash
# 补充角色权限
$ kubectl  edit role/argocd-application-controller -n argocd
...
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  - namespaces # <--- 在这里

# 确认角色绑定
$ kubectl get rolebinding -n argocd -o yaml

# 删除pod
$ kubectl delete pod -n argocd -l app.kubernetes.io/name=argocd-application-controller
```



## 8 Install operator-lifecycle-manager

```bash
# https://github.com/operator-framework/operator-lifecycle-manager/releases
# https://operatorhub.io/operator/argocd-operator

cd 
mkdir -p operator-lifecycle-manager && cd operator-lifecycle-manager

# upload files. 

root@master2:~/operator-lifecycle-manager# ls -lh 
total 413M
-rw-r--r-- 1 root root  222 Dec 12 17:50 argocd-operator.yaml
-rw-r--r-- 1 root root 159M Jan 16 21:27 catalog.tar
-rw-r--r-- 1 root root 807K Dec 12 17:25 crds.yaml
-rw-r--r-- 1 root root 1.9K Dec 12 17:25 install.sh
-rw-r--r-- 1 root root 127M Jan 16 21:28 olm.tar
-rw-r--r-- 1 root root  11K Jan 16 21:18 olm.yaml
-rw-r--r-- 1 root root  11K Jan 16 21:17 olm.yaml.bak
-rw-r--r-- 1 root root 127M Dec 12 17:26 operator-lifecycle-manager_0.26.0_linux_amd64.tar.gz



# edit config
cp  olm.yaml{,.bak}
vim olm.yaml
# 替换镜像为本地仓库的
root@master1:~/operator-lifecycle-manager# grep image: olm.yaml
          image: harbor.dockerregistry.com/kubernetes/operator-framework/olm:v0.26 
          image: harbor.dockerregistry.com/kubernetes/operator-framework/olm:v0.26 
                image: harbor.dockerregistry.com/kubernetes/operator-framework/olm:v0.26 
  image: harbor.dockerregistry.com/kubernetes/operatorhubio/catalog:latest

# install 
kubectl create -f crds.yaml 
kubectl create -f olm.yaml 


#  如果不行，就使用离线镜像
kubectl delete -f olm.yaml 
kubectl get namespace olm -o json > olm-temp.json
kubectl get namespace operators -o json > operators-temp.json
vim olm-temp.json 
vim operators-temp.json
kubectl replace --raw "/api/v1/namespaces/olm/finalize" -f ./olm-temp.json
kubectl replace --raw "/api/v1/namespaces/operators/finalize" -f ./operators-temp.json

docker load -i catalog.tar
docker load -i olm.tar
rm olm.yaml -f 
mv olm.yaml.bak olm.yaml



# 再次 install 
kubectl create -f olm.yaml 
```



## 9 Install operator

```bash
# install operator
root@master1:~/operator-lifecycle-manager# kubectl create -f argocd-operator.yaml 
subscription.operators.coreos.com/my-argocd-operator created

# view csv
root@master1:~/operator-lifecycle-manager# kubectl get csv -n operators
NAME                     DISPLAY   VERSION   REPLACES                 PHASE
argocd-operator.v0.8.0   Argo CD   0.8.0     argocd-operator.v0.7.0   Installing

# 等待2min
root@master1:~/operator-lifecycle-manager# kubectl get csv -n operators
NAME                     DISPLAY   VERSION   REPLACES                 PHASE
argocd-operator.v0.8.0   Argo CD   0.8.0     argocd-operator.v0.7.0   Succeeded

# view pod
root@master1:~/operator-lifecycle-manager# kubectl get pod -n operators 
NAME                                                  READY   STATUS    RESTARTS      AGE
argocd-operator-controller-manager-58876c86fd-z4b8d   1/1     Running   1 (55s ago)   95s


# set default # 不太建议
# kubectl config set-context --current --namespace=default
```



## 10 ADD DNS

```SHELL
root@master1:~/operator-lifecycle-manager# kubectl -n kube-system edit cm coredns
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        hosts {
          10.1.1.110  harbor.dockerregistry.com
          10.1.1.117  argocd.k8scluster.com     ### 加一条
          fallthrough
        }
...

# 重启dns
root@master1:~/operator-lifecycle-manager# kubectl delete -n kube-system pod `kubectl get pod -A|grep dns|awk '{print $2}'`
```





## 11 ADD LOCAL USER

```shell
# https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/#tying-it-all-together
# https://blog.csdn.net/sD7O95O/article/details/125966677

root@master2:~/gitlab-runner# kubectl edit -n argocd cm argocd-rbac-cm
configmap/argocd-rbac-cm edited

root@master2:~/gitlab-runner# kubectl edit -n argocd cm argocd-cm
configmap/argocd-cm edited
root@master2:~/gitlab-runner# echo y | argocd login 10.1.1.117:31812 --password 'T9zMQf5sME1buMJCM68W0Nx3R' --username admin
WARNING: server certificate had error: tls: failed to verify certificate: x509: cannot validate certificate for 10.1.1.117 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? 'admin:login' logged in successfully
Context '10.1.1.117:31812' updated
root@master2:~/gitlab-runner# argocd account update-password \
 --account db-admins \
 --current-password 'T9zMQf5sMm16GbU7E1buMJCM68W0Nx3R' \
 --new-password '870TJ6OVGNcLtMEE0p3SAvekRj9pnbiS'
Password updated
root@master2:~/gitlab-runner# echo y | argocd login 10.1.1.117:31812 --password '870TJ6OVGNcAvekRj9pnbiS' --username db-admins
WARNING: server certificate had error: tls: failed to verify certificate: x509: cannot validate certificate for 10.1.1.117 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? 'db-admins:login' logged in successfully
Context '10.1.1.117:31812' updated
root@master2:~/gitlab-runner# argocd account list
NAME       ENABLED  CAPABILITIES
admin      true     login
db-admins  true     apiKey, login 
root@master2:~/gitlab-runner# argocd app list
NAME             CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                           PATH               TARGET
argocd/frame-ui  https://kubernetes.default.svc             default  Synced  Healthy  <none>      <none>      http://10.1.1.11:5010/rdcenter/frame-ui.git    k8s/overlays/prod  test
```



> 角色权限部分，这里的`default` 对应Argo 中的项目概念，即默认项目。



```shell
root@master2:~/gitlab-runner# kubectl get -n argocd cm argocd-rbac-cm -oyaml
apiVersion: v1
data:  # <- 整个data 语法块代码
  policy.csv: |
    p, role:staging-db-admin, applications, create, default/*, allow
    p, role:staging-db-admin, applications, delete, default/*, allow
    p, role:staging-db-admin, applications, get, default/*, allow
    p, role:staging-db-admin, applications, override, default/*, allow
    p, role:staging-db-admin, applications, sync, default/*, allow
    p, role:staging-db-admin, applications, update, default/*, allow
    p, role:staging-db-admin, logs, get, default/*, allow
    p, role:staging-db-admin, exec, create, default/*, allow
    p, role:staging-db-admin, projects, get, default, allow
    g, db-admins, role:staging-db-admin
  policy.default: role:readonly
kind: ConfigMap
metadata:
  creationTimestamp: "2024-01-16T13:06:31Z"
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-rbac-cm
  namespace: argocd
  resourceVersion: "303385"
  uid: c7cf48bf-ad22-4dfd-b3a8-17b17719566f
  
#################################################################################### 关键部分
data:  
  policy.csv: |
    p, role:staging-db-admin, applications, create, default/*, allow
    p, role:staging-db-admin, applications, delete, default/*, allow
    p, role:staging-db-admin, applications, get, default/*, allow
    p, role:staging-db-admin, applications, override, default/*, allow
    p, role:staging-db-admin, applications, sync, default/*, allow
    p, role:staging-db-admin, applications, update, default/*, allow
    p, role:staging-db-admin, logs, get, default/*, allow
    p, role:staging-db-admin, exec, create, default/*, allow
    p, role:staging-db-admin, projects, get, default, allow
    g, db-admins, role:staging-db-admin
  policy.default: role:readonly
  
################################################################################################################################
# 不同用户组管理自己的项目
root@master2:~/gitlab-runner# kubectl get -n argocd cm argocd-rbac-cm -oyaml
apiVersion: v1
data:
  policy.csv: |
    p, role:staging-db-admin, applications, create, default/*, allow
    p, role:staging-db-admin, applications, delete, default/*, allow
    p, role:staging-db-admin, applications, get, default/*, allow
    p, role:staging-db-admin, applications, override, default/*, allow
    p, role:staging-db-admin, applications, sync, default/*, allow
    p, role:staging-db-admin, applications, update, default/*, allow
    p, role:staging-db-admin, logs, get, default/*, allow
    p, role:staging-db-admin, exec, create, default/*, allow
    p, role:staging-db-admin, projects, get, default, allow
    g, db-admins, role:staging-db-admin
    p, role:papd-bom-admin, applications, create, papd-bom/*, allow  ## 继续新增即可
    p, role:papd-bom-admin, applications, delete, papd-bom/*, allow
    p, role:papd-bom-admin, applications, get, papd-bom/*, allow
    p, role:papd-bom-admin, applications, override, papd-bom/*, allow
    p, role:papd-bom-admin, applications, sync, papd-bom/*, allow
    p, role:papd-bom-admin, applications, update, papd-bom/*, allow
    p, role:papd-bom-admin, logs, get, papd-bom/*, allow
    p, role:papd-bom-admin, exec, create, papd-bom/*, allow
    p, role:papd-bom-admin, projects, get, papd-bom, allow
    g, papd-bom, role:papd-bom-admin
  policy.default: role:readonly
kind: ConfigMap
metadata:
  creationTimestamp: "2024-01-16T13:06:31Z"
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-rbac-cm
  namespace: argocd
  resourceVersion: "336161"
  uid: c7cf48bf-ad22-4dfd-b3a8-17b17719566f
```



> 用户部分



```shell
root@master2:~/gitlab-runner# kubectl get -n argocd cm argocd-cm -oyaml
apiVersion: v1
data:   # <- 整个data 语法块代码
  accounts.db-admins: apiKey,login
kind: ConfigMap
metadata:
  creationTimestamp: "2024-01-16T13:06:30Z"
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
  resourceVersion: "304731"
  uid: 22f2f02d-a813-495a-98f1-53dab33214c3
  
 #################################################################################### 关键部分
 data:
  accounts.db-admins: apiKey,login
```

