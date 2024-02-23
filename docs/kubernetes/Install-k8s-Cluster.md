## 1 Create Config

```bash
# on master1 

# https://kubernetes.io/zh-cn/docs/reference/config-api/kubeadm-config.v1beta3/
# print default config
$ kubeadm config print init-defaults > kubeadm-init.yaml

# edit it(ipaddr) for your own needs.
cat <<EOF | tee  kubeadm-init.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.171.136
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/control-plane
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  external:
    endpoints:
      - https://192.168.171.136:2379
      - https://192.168.171.137:2379
      - https://192.168.171.138:2379
    caFile: /etc/kubernetes/pki/etcd/etcd-ca.pem
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.pem
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client-key.pem
kubernetesVersion: 1.27.6
imageRepository: registry.aliyuncs.com/google_containers
#controlPlaneEndpoint: 192.168.171.135:6443
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
EOF
```



## 2 Install k8s

```bash
# on master1

# begin to init k8s cluster
$ kubeadm init --config=kubeadm-init.yaml --upload-certs
...

# on lb1 and lb2 
systemctl restart haproxy.service 



# $ kubeadm init --config=kubeadm-init.yaml --upload-certs
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.171.136:6443 --token 64i2i5.vtdkoz3he8cttwoe \
        --discovery-token-ca-cert-hash sha256:9df932af344252c2b25cba8e2a1096c95041f64ec0f99c57c5f87be489639584 \
        --control-plane --certificate-key 195651292db2a759a52199cd0c9f22f0665864cfe92672ddf9391b5e4503e7e7

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.171.136:6443 --token 64i2i5.vtdkoz3he8cttwoe \
        --discovery-token-ca-cert-hash sha256:9df932af344252c2b25cba8e2a1096c95041f64ec0f99c57c5f87be489639584 
```



## 3 Reset node

```bash
# !!!Note: reset node if need
kubeadm reset

# !!!Note: clean dir if need
rm -rf /etc/cni/net.d
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm --clear
rm -rf $HOME/.kube


# scp ca again 
mkdir -p /etc/kubernetes/pki/etcd/
scp -r /etc/etcd/ssl/etcd-ca.pem    master1:/etc/kubernetes/pki/etcd/
scp -r /etc/etcd/ssl/client.pem     master1:/etc/kubernetes/pki/apiserver-etcd-client.pem
scp -r /etc/etcd/ssl/client-key.pem master1:/etc/kubernetes/pki/apiserver-etcd-client-key.pem

# clean etcd data
# on all etcd nodes.
systemctl stop etcd

# on all etcd nodes.
rm /data/etcd/* -rf

## after all node data clean !!!!
# on all etcd nodes.
systemctl start etcd

# check healthy
etcdctl endpoint health -w table
```



## 4 Add  node

```bash
# add master node

  kubeadm join 192.168.171.136:6443 --token 64i2i5.vtdkoz3he8cttwoe \
        --discovery-token-ca-cert-hash sha256:9df932af344252c2b25cba8e2a1096c95041f64ec0f99c57c5f87be489639584 \
        --control-plane --certificate-key 195651292db2a759a52199cd0c9f22f0665864cfe92672ddf9391b5e4503e7e7 \
        --cri-socket=unix:///var/run/containerd/containerd.sock

echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
source  /etc/profile

# add worker node
kubeadm join 192.168.171.136:6443 --token 64i2i5.vtdkoz3he8cttwoe \
        --discovery-token-ca-cert-hash sha256:9df932af344252c2b25cba8e2a1096c95041f64ec0f99c57c5f87be489639584 \
        --cri-socket=unix:///var/run/containerd/containerd.sock

echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile
source  /etc/profile


# 重新上传证书
# https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/#uploading-control-plane-certificates-to-the-cluster
kubeadm init phase upload-certs --upload-certs --config=/root/kubeadm-init.yaml 
# 重新生成token
kubeadm token create --print-join-command
# 清空节点证书目录
rm /etc/kubernetes/pki/* -rf 
# 加入主节点
kubeadm join 10.1.1.110:6443 --token 5zshml.7fppf46erf1jju9j --discovery-token-ca-cert-hash sha256:7cdaf0a19f5f02268eed7b39ebdf6c19a3ad163f093ae1183076014d55457be2 --control-plane --certificate-key 88f5cb76ca1e69207583ce39114bbf9f6b768dba456239c83d44126c7d0c2f50 --cri-socket=unix:///var/run/containerd/containerd.sock
```



## 5 Add label

```bash
# add worker label
 kubectl label nodes worker1.k8scluster.com node-role.kubernetes.io/worker=

```



## 6 Debug node

```bash
# dry-run
kubeadm init --config=kubeadm-init.yaml --dry-run


# And run init step again.
kubeadm init --config=kubeadm-init.yaml

journalctl -xeu kubelet -f 

# local test used , be careful.
kubectl taint nodes --all node-role.kubernetes.io/control-plane-


# test cluster
kubectl get pods -A
kubectl get cs
kubectl cluster-info
kubectl get nodes  # NoteReady is normal.
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
kubectl get componentstatuses

# add taint if not exited.
kubectl taint nodes master1.k8scluster.com node-role.kubernetes.io/control-plane=:NoSchedule
# del taint if exited.
kubectl taint nodes master1.k8scluster.com node-role.kubernetes.io/master-
```



## 7 Helm install calico

```bash
systemctl stop apparmor
systemctl disable apparmor 
systemctl restart containerd.service kubelet 

################################################################################################
# install used helm .
# https://github.com/projectcalico/calico/releases
# https://docs.tigera.io/calico/latest/getting-started/kubernetes/helm
# https://docs.tigera.io/calico/latest/getting-started/kubernetes/helm#install-calico

helm repo add projectcalico https://docs.tigera.io/calico/charts

helm search repo projectcalico/tigera-operator -l 

mkdir -p calico && cd calico
# helm pull projectcalico/tigera-operator --version v3.26.3
# upload tgz file.

tar xf tigera-operator-v3.26.3.tgz 
cp tigera-operator/values.yaml{,.bak}


# on all nodes.
# upload offline-images dir.
for image in offline-images/*.tar; do   docker load -i $image; done

# modify config
vim  tigera-operator/values.yaml
imagePullSecrets: {}

installation:
  enabled: true
  kubernetesProvider: ""
  calicoNetwork: ## here 
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: IPIP
      natOutgoing: Enabled
      nodeSelector: all()



kubectl create namespace tigera-operator
# helm install calico projectcalico/tigera-operator --version v3.24.6 -f values.yaml --namespace tigera-operator


helm install calico ./tigera-operator --namespace tigera-operator -f tigera-operator/values.yaml


# view pod and nodes status
root@master1:~/calico# kubectl get po -A -owide 
NAMESPACE          NAME                                             READY   STATUS    RESTARTS        AGE   IP             NODE                     NOMINATED NODE   READINESS GATES
calico-apiserver   calico-apiserver-577cb78868-b628k                1/1     Running   0               18s   10.244.113.2   master2.k8scluster.com   <none>           <none>
calico-apiserver   calico-apiserver-577cb78868-ljlt6                1/1     Running   0               11m   10.244.46.2    master3.k8scluster.com   <none>           <none>
calico-system      calico-kube-controllers-dcf6df9f9-sgrsm          1/1     Running   0               36m   10.244.208.2   master1.k8scluster.com   <none>           <none>
calico-system      calico-node-cxz7j                                1/1     Running   0               12m   192.168.1.128      master3.k8scluster.com   <none>           <none>
calico-system      calico-node-kgxvl                                1/1     Running   0               12m   192.168.1.127      master2.k8scluster.com   <none>           <none>
calico-system      calico-node-mm8mv                                1/1     Running   0               12m   192.168.1.129      worker1.k8scluster.com   <none>           <none>
calico-system      calico-node-zx84b                                1/1     Running   0               13m   192.168.1.126      master1.k8scluster.com   <none>           <none>
calico-system      calico-typha-6674977c9-gmbbv                     1/1     Running   0               28m   192.168.1.127      master2.k8scluster.com   <none>           <none>
calico-system      calico-typha-6674977c9-vqm5p                     1/1     Running   0               36m   192.168.1.128      master3.k8scluster.com   <none>           <none>
calico-system      csi-node-driver-bxjp9                            2/2     Running   0               36m   10.244.208.3   master1.k8scluster.com   <none>           <none>
calico-system      csi-node-driver-kbh6d                            2/2     Running   0               36m   10.244.85.65   worker1.k8scluster.com   <none>           <none>
calico-system      csi-node-driver-phhjj                            2/2     Running   0               36m   10.244.46.1    master3.k8scluster.com   <none>           <none>
calico-system      csi-node-driver-v4dkb                            2/2     Running   0               36m   10.244.113.1   master2.k8scluster.com   <none>           <none>
kube-system        coredns-7bdc4cb885-htf8t                         1/1     Running   0               59m   10.244.208.4   master1.k8scluster.com   <none>           <none>
kube-system        coredns-7bdc4cb885-s4b9j                         1/1     Running   0               59m   10.244.208.1   master1.k8scluster.com   <none>           <none>
kube-system        kube-apiserver-master1.k8scluster.com            1/1     Running   0               80m   192.168.1.126      master1.k8scluster.com   <none>           <none>
kube-system        kube-apiserver-master2.k8scluster.com            1/1     Running   0               65m   192.168.1.127      master2.k8scluster.com   <none>           <none>
kube-system        kube-apiserver-master3.k8scluster.com            1/1     Running   0               65m   192.168.1.128      master3.k8scluster.com   <none>           <none>
kube-system        kube-controller-manager-master1.k8scluster.com   1/1     Running   3 (49m ago)     80m   192.168.1.126      master1.k8scluster.com   <none>           <none>
kube-system        kube-controller-manager-master2.k8scluster.com   1/1     Running   1 (25m ago)     65m   192.168.1.127      master2.k8scluster.com   <none>           <none>
kube-system        kube-controller-manager-master3.k8scluster.com   1/1     Running   1 (2m48s ago)   65m   192.168.1.128      master3.k8scluster.com   <none>           <none>
kube-system        kube-proxy-95f92                                 1/1     Running   0               66m   192.168.1.127      master2.k8scluster.com   <none>           <none>
kube-system        kube-proxy-cf7rl                                 1/1     Running   0               65m   192.168.1.128      master3.k8scluster.com   <none>           <none>
kube-system        kube-proxy-jv5vb                                 1/1     Running   0               65m   192.168.1.129      worker1.k8scluster.com   <none>           <none>
kube-system        kube-proxy-tj82v                                 1/1     Running   0               80m   192.168.1.126      master1.k8scluster.com   <none>           <none>
kube-system        kube-scheduler-master1.k8scluster.com            1/1     Running   3 (49m ago)     80m   192.168.1.126      master1.k8scluster.com   <none>           <none>
kube-system        kube-scheduler-master2.k8scluster.com            1/1     Running   0               65m   192.168.1.127      master2.k8scluster.com   <none>           <none>
kube-system        kube-scheduler-master3.k8scluster.com            1/1     Running   2 (2m43s ago)   65m   192.168.1.128      master3.k8scluster.com   <none>           <none>
tigera-operator    tigera-operator-f6bb878c4-zrt8n                  1/1     Running   2 (2m49s ago)   36m   192.168.1.129      worker1.k8scluster.com   <none>           <none>
root@master1:~/calico# kubectl get nodes 
NAME                     STATUS   ROLES           AGE   VERSION
master1.k8scluster.com   Ready    control-plane   80m   v1.27.6
master2.k8scluster.com   Ready    control-plane   66m   v1.27.6
master3.k8scluster.com   Ready    control-plane   65m   v1.27.6
worker1.k8scluster.com   Ready    worker          65m   v1.27.6


# clean env 
docker system prune -af



##################################################################################################### 
# or install uesd yaml file. ## 但不太建议
# https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#install-calico
#kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
kubectl get all -o wide -n tigera-operator

#kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml
sed -i 's/cidr: 192.168.0.0/cidr: 10.244.0.0/g' custom-resources.yaml
sed -i 's/encapsulation: VXLANCrossSubnet/encapsulation: IPIP/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-


#####################################################################################################
# !!! Note: uninstall calico if need.
# helm uninstall calico -n tigera-operator
```



## 8 Add Harbor dns

```bash
root@master1:~/StorageClass# kubectl -n kube-system edit cm coredns
configmap/coredns edited


root@master1:~/StorageClass# kubectl -n kube-system get cm coredns -oyaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        hosts {
          10.1.1.2  harbor.dockerregistry.com
          fallthrough
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2023-12-04T08:00:44Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "216930"
  uid: 767269af-1974-41d6-8cd3-c0b28d92b97c

root@master1:~/StorageClass# kubectl delete -n kube-system pod `kubectl get pod -A|grep dns|awk '{print $2}'`
pod "coredns-7bdc4cb885-htf8t" deleted
pod "coredns-7bdc4cb885-s4b9j" deleted
```



## 9 Add taint

```bash
# 开启主节点的污点，不让业务容器跑过来

# 安装jq
apt-get update
apt-get install jq

# 查看污点
root@master1:~/gitlab-runner# kubectl get nodes -o json | jq -r '.items[] | {name: .metadata.name, taints: .spec.taints}'
{
  "name": "master1.k8scluster.com",
  "taints": null
}
{
  "name": "master2.k8scluster.com",
  "taints": null
}
{
  "name": "master3.k8scluster.com",
  "taints": null
}
{
  "name": "worker1.k8scluster.com",
  "taints": null
}

# 添加污点
# kubectl taint nodes <node-name> key=value:NoSchedule
kubectl taint nodes master1.k8scluster.com node-role.kubernetes.io/master=:NoSchedule
kubectl taint nodes master2.k8scluster.com node-role.kubernetes.io/master=:NoSchedule
kubectl taint nodes master3.k8scluster.com node-role.kubernetes.io/master=:NoSchedule

# 移除污点（若需要）
# kubectl taint nodes <node-name> key=value:NoSchedule-


# 查看污点
root@master1:~/gitlab-runner# kubectl get nodes -o json | jq -r '.items[] | {name: .metadata.name, taints: .spec.taints}'
{
  "name": "master1.k8scluster.com",
  "taints": [
    {
      "effect": "NoSchedule",
      "key": "node-role.kubernetes.io/master"
    }
  ]
}
{
  "name": "master2.k8scluster.com",
  "taints": [
    {
      "effect": "NoSchedule",
      "key": "node-role.kubernetes.io/master"
    }
  ]
}
{
  "name": "master3.k8scluster.com",
  "taints": [
    {
      "effect": "NoSchedule",
      "key": "node-role.kubernetes.io/master"
    }
  ]
}
{
  "name": "worker1.k8scluster.com",
  "taints": null
}
```



## 10 Expose Service Port

```shell
# all master nodes

root@master1:~# vim /etc/kubernetes/manifests/kube-scheduler.yaml
root@master1:~# grep "0.0.0.0" /etc/kubernetes/manifests/kube-scheduler.yaml
    - --bind-address=0.0.0.0  # here

  
  
root@master1:~# vim /etc/kubernetes/manifests/kube-controller-manager.yaml
root@master1:~# grep "0.0.0.0" /etc/kubernetes/manifests/kube-controller-manager.yaml
    - --bind-address=0.0.0.0 # here

```



## 11 Uninstall Calico

> !!! 高危操作

```shell
# 如果有必要的话
kubectl delete daemonset calico-node -n calico-system
kubectl delete deployment calico-kube-controllers -n calico-system
kubectl delete deployment calico-typha -n calico-system
kubectl delete service -l k8s-app=calico -n calico-system
kubectl delete configmap -l k8s-app=calico -n calico-system
kubectl delete serviceaccount -l k8s-app=calico -n calico-system
kubectl delete clusterrole,clusterrolebinding -l k8s-app=calico
kubectl delete crd -l k8s-app=calico
kubectl delete namespace calico-system

kubectl delete pod calico-kube-controllers-dcf6df9f9-sgrsm -n calico-system --force --grace-period=0
kubectl delete pod csi-node-driver-8xw8r -n calico-system --force --grace-period=0
kubectl delete pod csi-node-driver-k28g4 -n calico-system --force --grace-period=0
kubectl delete pod csi-node-driver-kbh6d -n calico-system --force --grace-period=0
kubectl delete pod csi-node-driver-qvl6h -n calico-system --force --grace-period=0
kubectl delete pod csi-node-driver-sm6q5 -n calico-system --force --grace-period=0
kubectl delete pod csi-node-driver-zpqxh -n calico-system --force --grace-period=0



kubectl get namespace calico-apiserver -o json > calico-apiserver-temp.json
kubectl get namespace calico-system -o json > calico-system-temp.json
vim calico-apiserver-temp.json 
vim calico-system-temp.json
kubectl replace --raw "/api/v1/namespaces/calico-apiserver/finalize" -f ./calico-apiserver-temp.json
kubectl replace --raw "/api/v1/namespaces/calico-system/finalize" -f ./calico-system-temp.json

modprobe -r ipip
rm /etc/cni/net.d/* -rf 
rm /var/lib/cni/ -rf
systemctl restart kubelet containerd
kubectl delete -n kube-system pod `kubectl get pod -A|grep dns|awk '{print $2}'`
sync;reboot
```





## 12 Reset Cluster

```shell
kubeadm reset

iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -P FORWARD ACCEPT

ipvsadm --clear
ip link delete kube-ipvs0


rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet/
rm -rf /var/lib/etcd/
rm -rf /var/lib/cni/
rm -f $HOME/.kube/config


ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
modprobe -r ipip
rm /etc/cni/net.d/* -rf 
rm /var/lib/cni/ -rf

rm -rf /etc/cni/net.d/

systemctl stop apparmor
systemctl disable apparmor 
systemctl restart containerd.service

systemctl stop etcd
rm /data/etcd/* -rf 
rm /data/etcd-backups/backup-202401*.db -rf 
rm /data/etcd-backups/backup-202401*.log -rf 
systemctl start etcd


```

