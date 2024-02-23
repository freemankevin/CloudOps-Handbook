

## 1 部署`openelb`

[官方文档](https://openelb.io/docs/getting-started/installation/)

```shell


wget https://raw.githubusercontent.com/openelb/openelb/master/deploy/openelb.yaml
# 修改镜像为本地或国内的，否则会报错
$ cat openelb.yaml| grep image:
        image: harbor.dockerregistry.com/gitops/kubesphere/openelb:v0.5.1
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1
        
$ kubectl apply -f openelb.yaml

# 若本地443端口已使用导致报错，请修改deployment 里的端口443 为其他的，如7443

# 若DaemonSets 的镜像拉取超时，可以换成本地的
          image: 'harbor.dockerregistry.com/gitops/kubesphere/kube-keepalived-vip:0.35'

```


## 2  部署`lay2-eip`

[官方文档](https://openelb.io/docs/getting-started/usage/use-openelb-in-layer-2-mode/)

```shell

$ vim layer2-eip.yaml
apiVersion: network.kubesphere.io/v1alpha2
kind: Eip
metadata:
    name: layer2-eip
    annotations:
      eip.openelb.kubesphere.io/is-default-eip: "true"
spec:
    address: 10.1.1.10-10.1.1.20
    protocol: layer2
    interface: ens192
    disable: false

kubectl apply -f layer2-eip.yaml


## 为 kube-proxy 启用 strictARP
$ kubectl edit configmap kube-proxy -n kube-system
## 修改false 为true
ipvs:
  strictARP: true
$ kubectl rollout restart daemonset kube-proxy -n kube-system


## svc elb 配置时添加注释：
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: layer2-eip
```


## 3 部署应用

[官方文档](https://openelb.io/docs/getting-started/usage/use-openelb-in-layer-2-mode/)


```shell
$ vi layer2-svc.yaml
kind: Service
apiVersion: v1
metadata:
  name: layer2-svc
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: layer2-eip
spec:
  selector:
    app: layer2-openelb
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 8080
  externalTrafficPolicy: Cluster

$ kubectl apply -f layer2-svc.yaml





$ vi layer2-openelb-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: layer2-openelb
spec:
  replicas: 2
  selector:
    matchLabels:
      app: layer2-openelb
  template:
    metadata:
      labels:
        app: layer2-openelb
    spec:
      containers:
        - image: harbor.dockerregistry.com/gitops/luksa/kubia
          name: kubia
          ports:
            - containerPort: 8080

$ kubectl apply -f layer2-openelb-deploy.yaml



$ kubectl get svc

$ curl `kubectl get svc | grep layer| awk '{print $4}'`

```