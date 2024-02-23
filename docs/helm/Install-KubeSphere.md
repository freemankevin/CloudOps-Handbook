## 1 Install kubesphere

```bash
# https://github.com/kubesphere/kubesphere
# https://kubesphere.io/zh/docs/v3.4/quick-start/minimal-kubesphere-on-k8s/

# on all k8s nodes.
# kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.4.1/kubesphere-installer.yaml
# kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.4.1/cluster-configuration.yaml

# add zone
export KKZONE=cn
echo "export KKZONE=cn" >>/etc/profile
source /etc/profile


# upload files.
root@master1:~# mkdir -p kubesphere  && cd kubesphere/
root@master1:~/kubesphere# ls -lh 
total 1.1G
-rw-r--r-- 1 root root  11K Dec  5 10:40 cluster-configuration.yaml
-rw-r--r-- 1 root root 1.1G Nov 24 17:44 images-dir.tar.gz
-rw-r--r-- 1 root root 4.6K Dec  5 10:39 kubesphere-installer.yaml

# unpack images file.
root@master1:~/kubesphere# tar xf images-dir.tar.gz 
root@master1:~/kubesphere# for img in $(ls images-dir/*.tar); do docker load -i $img; done
unpacking docker.io/prom/alertmanager:v0.23.0 (sha256:323cbba39f5a4df7c750ead201eed8d1f96cfd04606984059eb90b7702a12d42)...
Loaded image: prom/alertmanager:v0.23.0
unpacking registry.cn-beijing.aliyuncs.com/kubesphereio/cni:v3.26.1 (sha256:dbdd8749b4d394abf14b528fbc5a41d654953c7e4d08f1a2133e9300affe0e98)...
Loaded image: registry.cn-beijing.aliyuncs.com/kubesphereio/cni:v3.26.1
unpacking registry.cn-beijing.aliyuncs.com/kubesphereio/coredns:1.8.6 (sha256:7aba92733295a77c554e143d7a625c5cad620944b5bce174faeb512f9c61c277)...
Loaded image: registry.cn-beijing.aliyuncs.com/kubesphereio/coredns:1.8.6
unpacking docker.io/mirrorgooglecontainers/defaultbackend-amd64:1.4 (sha256:df51dfa3712efbbc6c3f05352c5c2be2e8e1a0bfe3eae5b915101a5a1ea67242)...
Loaded image: mirrorgooglecontainers/defaultbackend-amd64:1.4
...


# on master1.
# install kubesphere
root@master1:~/kubesphere# kubectl create -f kubesphere-installer.yaml
customresourcedefinition.apiextensions.k8s.io/clusterconfigurations.installer.kubesphere.io created
namespace/kubesphere-system created
serviceaccount/ks-installer created
clusterrole.rbac.authorization.k8s.io/ks-installer created
clusterrolebinding.rbac.authorization.k8s.io/ks-installer created
deployment.apps/ks-installer created

root@master1:~/kubesphere# kubectl create -f cluster-configuration.yaml
clusterconfiguration.installer.kubesphere.io/ks-installer created



# view logs
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f  # 如果查看不到就看POD

kubectl get po -A # 查看pod 状态




# chrome open it .
Console: http://192.168.0.2:30880 # ip 换成自己的
Account: admin
Password: P@88w0rd
```



## 2 Uninstall kubesphere

> !!! 高危操作

```shell
# https://kubesphere.io/zh/docs/v3.3/installing-on-kubernetes/uninstall-kubesphere-from-k8s/
# https://blog.csdn.net/o0haidee0o/article/details/116745171
# https://github.com/kubesphere/ks-installer/blob/v3.4.1/scripts/kubesphere-delete.sh
chmod +x kubesphere-delete.sh 
./kubesphere-delete.sh  # 执行结束后，会卡在删除命名空间那里


# 通过命令清理
kubectl get namespace kubesphere-controls-system -o json > kubesphere-controls-system-temp.json
vim kubesphere-controls-system-temp.json 
...
  "spec": {   
    "finalizers": [xxx] # 将这里全部删除，包括："finalizers":
  },   
...

kubectl replace --raw "/api/v1/namespaces/kubesphere-controls-system/finalize" -f ./kubesphere-controls-system-temp.json


kubectl get namespace kubesphere-system -o json > kubesphere-system-temp.json
vim kubesphere-system-temp.json 
vim kubesphere-controls-system-temp.json 
...
  "spec": {   
    "finalizers": [xxx] # 将这里全部删除，包括："finalizers":
  },   
...

kubectl replace --raw "/api/v1/namespaces/kubesphere-system/finalize" -f ./kubesphere-system-temp.json
```