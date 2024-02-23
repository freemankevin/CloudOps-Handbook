## 1 Install nfs server

```bash
# https://jimmysong.io/kubernetes-handbook/practice/using-nfs-for-persistent-storage.html
# https://github.com/kubernetes-retired/external-storage/tree/master/nfs-client/deploy

# intall package
apt-get update
apt-get install nfs-kernel-server -y

# mkdir & chown
mkdir -p /data/ifs/kubernetes
chown -R nobody:nogroup /data/ifs/kubernetes

# limit client ip 
echo "/data/ifs/kubernetes 10.1.1.0/32(no_root_squash,rw,sync,no_subtree_check)" | tee -a /etc/exports

# start svc
systemctl start nfs-kernel-server
systemctl enable nfs-kernel-server
# ufw allow from 10.1.1.0/32 to any port nfs
```



## 2 Install nfs client 

```bash
# on all k8s nodes.
apt-get update
apt-get install nfs-common -y

# on master1
echo -e "\n10.1.1.12 nfs-server.k8sCluster.com" >> /etc/hosts

# scp file.
scp -r /etc/hosts master2:/etc/
scp -r /etc/hosts master3:/etc/
scp -r /etc/hosts worker1:/etc/

# on all k8s nodes.
# test
root@master1:~/kubesphere# showmount -e nfs-server.k8sCluster.com
Export list for nfs-server.k8sCluster.com:
/data/ifs/kubernetes 10.1.1.0/32
```


## 3 Install nfs sc 

```bash
# https://v1-27.docs.kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/#nfs
# https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy

# on master1
mkdir -p StorageClass && cd StorageClass
# upload file.
root@master1:~/StorageClass# ll 
total 24
-rw-r--r-- 1 root root  246 Dec  5 16:14 class.yaml
-rw-r--r-- 1 root root 1094 Dec  5 16:26 deployment.yaml
-rw-r--r-- 1 root root   64 Dec  5 16:14 kustomization.yaml
-rw-r--r-- 1 root root 1900 Dec  5 16:15 rbac.yaml
-rw-r--r-- 1 root root  190 Dec  5 16:15 test-claim.yaml
-rw-r--r-- 1 root root  401 Dec  5 16:15 test-pod.yaml

# install sc
root@master1:~/StorageClass# kubectl create -f class.yaml 
storageclass.storage.k8s.io/nfs-client created

root@master1:~/StorageClass# kubectl create -f rbac.yaml 
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

root@master1:~/StorageClass# kubectl create -f deployment.yaml ### 修改里面服务器IP，以及共享路径
deployment.apps/nfs-client-provisioner created

# test sc
root@master1:~/StorageClass# kubectl create -f test-claim.yaml 
persistentvolumeclaim/test-claim created
root@master1:~/StorageClass# kubectl create -f test-pod.yaml 
pod/test-pod created
root@master1:~/StorageClass# kubectl get po 
NAME                                      READY   STATUS      RESTARTS      AGE
nfs-client-provisioner-7fddb5f6fd-ljmq9   1/1     Running     1 (82s ago)   3m24s
test-pod                                  0/1     Completed   0             4s
root@master1:~/StorageClass# kubectl get po 
NAME                                      READY   STATUS      RESTARTS      AGE
nfs-client-provisioner-7fddb5f6fd-ljmq9   1/1     Running     1 (84s ago)   3m26s
test-pod                                  0/1     Completed   0             6s
root@master1:~/StorageClass# kubectl get pvc 
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-d52e62c4-9c2d-415a-8404-61a812fa139e   1Mi        RWX            nfs-client     15s

# set default
root@master1:~/StorageClass# kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/nfs-client patched
root@master1:~/StorageClass# kubectl get sc
NAME                   PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  5m39s


# on nfs server .

root@harbor:~# ll /data/ifs/kubernetes/
total 12
drwxrwxrwx 2 root root 4096 Dec  5 16:33 default-test-claim-pvc-d52e62c4-9c2d-415a-8404-61a812fa139e
drwxrwxrwx 3 root root 4096 Dec  5 16:35 kubesphere-monitoring-system-prometheus-k8s-db-prometheus-k8s-0-pvc-bac1980f-5fe0-44a3-a5ae-c21d3e28f98c
drwxrwxrwx 3 root root 4096 Dec  5 16:35 kubesphere-monitoring-system-prometheus-k8s-db-prometheus-k8s-1-pvc-b7b004c6-d0cf-4126-ae63-49af5844cd0c
```

