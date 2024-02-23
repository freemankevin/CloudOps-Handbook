## 1 Office & others doc

```bash
# https://blog.csdn.net/m0_51510236/article/details/134142834#_kubeadmkubectlkubelet_500
# https://www.bilibili.com/video/BV1HN4y167fQ?p=6
# https://blog.csdn.net/lswzw/article/details/109027255
# https://gist.github.com/islishude/231659cec0305ace090b933ce851994a
# https://v1-27.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file
# https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network
# https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```



## 2 Change dns

```bash
# on master nodes.
hostnamectl set-hostname master1.k8sCluster.com
hostnamectl set-hostname master2.k8sCluster.com
hostnamectl set-hostname master3.k8sCluster.com

# on worker nodes.
hostnamectl set-hostname worker1.k8sCluster.com


# on master1 node.
# add dns 
echo -e "\n192.168.171.136  etcd1"      >> /etc/hosts
echo -e "192.168.171.50  harbor   harbor.dockerregistry.com"  >> /etc/hosts
echo -e "192.168.171.136  master1  master1.k8sCluster.com"  >> /etc/hosts
echo -e "192.168.171.137  master2  master1.k8sCluster.com"  >> /etc/hosts
echo -e "192.168.171.138  master3  master1.k8sCluster.com"  >> /etc/hosts
echo -e "192.168.171.139  worker1  worker1.k8sCluster.com"  >> /etc/hosts
# push public-key
# ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
sshpass -p 'a' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@harbor"
sshpass -p 'a' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@etcd1"
sshpass -p 'a' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@master1"
sshpass -p 'a' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@master2"
sshpass -p 'a' ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa.pub "root@worker1"
# scp files to others nodes.
scp -r /etc/hosts master2:/etc/
scp -r /etc/hosts master3:/etc/
scp -r /etc/hosts worker1:/etc/
# off swap
swapoff -a
sed -i '/swap/d' /etc/fstab
mount -a
ssh master2 "swapoff -a && sed -i '/swap/d' /etc/fstab && mount -a"
ssh master3 "swapoff -a && sed -i '/swap/d' /etc/fstab && mount -a"
ssh worker1 "swapoff -a && sed -i '/swap/d' /etc/fstab && mount -a"
```



## 3 Upload files

```bash
# on master1 

# upload files to /root
root@master1:~# ls -lh 
total 350M
-rw-r--r-- 1 root root  39M Oct 16 17:21 cni-plugins-linux-amd64-v1.2.0.tgz
-rw-r--r-- 1 root root  45M Oct 16 16:59 containerd-1.7.6-linux-amd64.tar.gz
-rw-r--r-- 1 root root 1.3K Oct 16 17:06 containerd.service
-rw-r--r-- 1 root root  16M Nov 17 10:44 helm-v3.12.3-linux-amd64.tar.gz
-rwxr-xr-x 1 root root 2.2K Dec  1 09:37 init-os-debian.sh
-rw-r--r-- 1 root root 241M Oct 16 17:52 nerdctl-full-1.6.2-linux-amd64.tar.gz
-rw-r--r-- 1 root root  11M Oct 16 17:16 runc.amd64


# add harbor & etcd ca files.

mkdir -p /etc/tls/harbor/     /etc/kubernetes/pki/etcd/
# upload ca file to localhost.
scp -r harbor:/etc/docker/certs.d/harbor.dockerregistry.com  /etc/tls/harbor/
scp -r etcd1:/etc/etcd/ssl/etcd-ca.pem    /etc/kubernetes/pki/etcd/
scp -r etcd1:/etc/etcd/ssl/client.pem     /etc/kubernetes/pki/apiserver-etcd-client.pem
scp -r etcd1:/etc/etcd/ssl/client-key.pem /etc/kubernetes/pki/apiserver-etcd-client-key.pem

root@master1:~# tree /etc/tls/harbor/
/etc/tls/harbor/
└── harbor.dockerregistry.com
    ├── ca.crt
    ├── harbor.dockerregistry.com.cert
    └── harbor.dockerregistry.com.key

1 directory, 3 files

root@master1:~# tree /etc/kubernetes/pki/
/etc/kubernetes/pki/
├── apiserver-etcd-client-key.pem
├── apiserver-etcd-client.pem
└── etcd
    └── etcd-ca.pem

1 directory, 3 files


# scp to others nodes.
scp -r ./* master2:/root/
scp -r ./* master3:/root/
scp -r ./* worker1:/root/

ssh master2 "mkdir -p /etc/tls/harbor/ /etc/kubernetes/pki/etcd/"
ssh master3 "mkdir -p /etc/tls/harbor/ /etc/kubernetes/pki/etcd/"
ssh worker1 "mkdir -p /etc/tls/harbor/ /etc/kubernetes/pki/etcd/"

scp -r  /etc/tls/harbor/* master2:/etc/tls/harbor/
scp -r  /etc/tls/harbor/* master3:/etc/tls/harbor/
scp -r  /etc/tls/harbor/* worker1:/etc/tls/harbor/

scp -r  /etc/kubernetes/pki/* master2:/etc/kubernetes/pki/
scp -r  /etc/kubernetes/pki/* master3:/etc/kubernetes/pki/
scp -r  /etc/kubernetes/pki/* worker1:/etc/kubernetes/pki/

# all nodes.
\cp -rvf /etc/tls/harbor/harbor.dockerregistry.com/ca.crt /usr/local/share/ca-certificates/
\cp -rvf /etc/tls/harbor/harbor.dockerregistry.com/harbor.dockerregistry.com.cert /usr/local/share/ca-certificates/

# 更新证书存储
update-ca-certificates

# 重启 运行时 服务
systemctl restart containerd
```



## 4 Install containerd

```bash
# on all nodes.

# https://github.com/containerd/containerd/blob/main/docs/getting-started.md
# https://github.com/containerd/containerd/releases
tar Cxzvf /usr/local containerd-1.7.6-linux-amd64.tar.gz

# https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
mkdir -p /usr/local/lib/systemd/system/
\cp -rvf containerd.service  /usr/local/lib/systemd/system/containerd.service 
systemctl daemon-reload
systemctl enable --now containerd

# product config
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

mkdir -p /data/containerd

vim /etc/containerd/config.toml
...
required_plugins = []
# change dir to /data
root = "/data/containerd"
state = "/run/containerd"
...

    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
...
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""
      # right here to add 
      # public images repo
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://docker.mirrors.ustc.edu.cn","http://hub-mirror.c.163.com","https://ustc-edu-cn.mirror.aliyuncs.com"]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
        endpoint = ["https://gcr.mirrors.ustc.edu.cn"]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
        endpoint = ["https://gcr.mirrors.ustc.edu.cn/google-containers/"]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
        endpoint = ["https://quay.mirrors.ustc.edu.cn"]

      # local images repo
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.dockerregistry.com"]
        endpoint = ["https://harbor.dockerregistry.com"]
        username = "robot$5m2wwklmotet"
        password = "zHQi7muaaqM2t3Cb4rcYnCYlJMddxA3b"
        [plugins."io.containerd.grpc.v1.cri".registry.tls]
          #insecure_skip_verify = false
          ca_file = "/etc/tls/harbor/harbor.dockerregistry.com/ca.crt"
          cert_file = "/etc/tls/harbor/harbor.dockerregistry.com/harbor.dockerregistry.com.cert"
          key_file = "/etc/tls/harbor/harbor.dockerregistry.com/harbor.dockerregistry.com.key"
...
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true

# on master1 
scp -r /etc/containerd/config.toml master2:/etc/containerd/
scp -r /etc/containerd/config.toml master3:/etc/containerd/
scp -r /etc/containerd/config.toml worker1:/etc/containerd/


# restart service
systemctl daemon-reload 
systemctl restart containerd
systemctl status  containerd
```



## 5 Install runc

```bash
# on all nodes.

# https://github.com/opencontainers/runc/releases
install -m 755 runc.amd64 /usr/local/sbin/runc
```



## 6 Install cni

```bash
# on all nodes.

# https://github.com/containernetworking/plugins/releases
#tar Cxzvf /usr/local/bin cni-plugins-linux-amd64-v1.2.0.tgz

mkdir -p /opt/cni/bin

tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz

ll /opt/cni/bin/dhcp


```



## 7 Install nerdctl

```bash
# on all nodes.

# https://github.com/containerd/nerdctl#install
tar Cxzvf /usr/local nerdctl-full-1.6.2-linux-amd64.tar.gz

# alias to docker
echo "alias docker='nerdctl --namespace k8s.io'"  >> /etc/profile
echo "alias docker-compose='nerdctl compose'"  >> /etc/profile
echo "alias crictl='crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock'" >> /etc/profile
echo "export ETCDETC_API=3"  >> /etc/profile
source  /etc/profile


# add config
mkdir -p /etc/nerdctl/
cat > /etc/nerdctl/nerdctl.toml << 'EOF'
namespace      = "k8s.io"
insecure_registry = true
cni_path = "/opt/cni/bin/"
EOF


# login harbor
root@master1:~# docker login harbor.dockerregistry.com -u robot\$5m2wwklmotet -p zHQi7muaaqM2t3Cb4rcYnCYlJMddxA3b
WARN[0000] WARNING! Using --password via the CLI is insecure. Use --password-stdin. 
WARN[0000] skipping verifying HTTPS certs for "harbor.dockerregistry.com" 
WARNING: Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

# add buildkit
\cp -rvf /usr/local/lib/systemd/system/buildkit.service /lib/systemd/system/
systemctl enable buildkit.service --now
systemctl start buildkit.service --now
systemctl status buildkit.service --now
```



## 8 open ipvs

```bash
# Forwarding IPv4 and letting iptables see bridged traffic 
# https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic


# on all nodes.


# ipvs 
tee -a /etc/modules-load.d/ipvs.conf << EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

# containerd
cat << EOF > /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF


modprobe overlay
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack

update-initramfs -u



# sysctl params required by setup, params persist across reboots
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
vm.swappiness=0
user.max_user_namespaces=28633
EOF

sysctl -p 

# Apply sysctl params without reboot
sysctl --system

# Verify that the br_netfilter
lsmod | grep br_netfilter
lsmod | grep overlay
lsmod | grep ip_vs
lsmod | grep ip_vs_rr
lsmod | grep ip_vs_wrr
lsmod | grep ip_vs_sh
lsmod | grep nf_conntrack

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```



## 9 Install net-tools

```bash
# on all nodes.


# net-tools (optional)
apt install ipset ipvsadm -y
```



## 10 Install helm 

```bash
# https://github.com/helm/helm/releases/tag/v3.12.3
# https://get.helm.sh/helm-v3.12.3-linux-amd64.tar.gz

# on all nodes.


tar xf helm-v3.12.3-linux-amd64.tar.gz 
mv linux-amd64/helm /usr/local/bin/
helm version 
```



## 11 Add soft repo

```bash
# on all nodes.

# add aliyun repo
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
add-apt-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
#curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
#echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
apt update
```



## 12 Stop apparmor 

```bash
# https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized


# on all nodes.

systemctl stop apparmor
systemctl disable apparmor 
systemctl restart containerd.service
```



## 13 Install client

```bash
# on all nodes.

# install kubelet/kubeadm/kubectl
# search version
apt list --all-versions kubelet | grep '1.27\..*'
apt list --all-versions kubeadm | grep '1.27\..*'
apt list --all-versions kubectl | grep '1.27\..*'

# install specified version 
apt  install -y kubectl=1.27.6-00
apt  install -y kubelet=1.27.6-00
apt  install -y kubeadm=1.27.6-00

# scan pkg version 
apt list --installed | grep kubectl 
apt list --installed | grep kubelet 
apt list --installed | grep kubeadm 

# scan version 
kubectl   version
kubelet --version
kubeadm   version

# enable for all hosts
systemctl enable kubelet


# add completion
source <(kubectl completion bash)
source <(kubeadm completion bash)

# hold version without update forever
apt-mark hold kubelet kubeadm kubectl


# !!!note： update version if need.
apt-mark unhold kubelet kubeadm kubectl
apt-get install -y kubelet=1.27.8-00 kubeadm=1.27.8-00 kubectl=1.27.8-00  # apt source.list must update to be latest.
apt-mark hold kubelet kubeadm kubectl

# !!!note： uninstall if need.
apt  autoremove -y kubectl
apt  autoremove -y kubelet
apt  autoremove -y kubeadm
```



## 14 Add auto-completion

```bash
# on all nodes.

# command auto-completion
apt update
apt install bash-completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
```



## 15 Pull images

```bash
# on all nodes.

# list & pull images
kubeadm config images list --kubernetes-version=v1.27.6
# kubeadm config images pull 
kubeadm config images pull --kubernetes-version=v1.27.6 --image-repository='registry.aliyuncs.com/google_containers'
```



## 16 Reboot

```bash
# on all nodes
sync;reboot
```

