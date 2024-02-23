## 1 Premise

```bash
# helm 要求：helm > 3.9 

root@master1:~# helm version 
version.BuildInfo{Version:"v3.12.3", GitCommit:"3a31588ad33fe3b89af5a2a54ee1d25bfe6eaa5e", GitTreeState:"clean", GoVersion:"go1.20.7"}
```



## 2 Upload files

```bash
# 添加helm仓库
# helm repo add traefik https://traefik.github.io/charts
# helm repo update
# helm search repo traefik/traefik -l # 罗列版本
# helm pull traefik/traefik --version 25.0.0 

# git clone -b v25.0.0 https://github.com/traefik/traefik-helm-chart.git

# upload files
mkdir -p traefik && cd  traefik
root@master1:~/traefik# ls -lh 
total 4.0K
drwxr-xr-x 4 root root 4.0K Dec  6 16:39 traefik-helm-chart
root@master1:~/traefik# cd traefik-helm-chart/
root@master1:~/traefik/traefik-helm-chart# ls -lh 
total 56K
-rw-r--r-- 1 root root  886 Dec  6 16:36 CONTRIBUTING.md
-rw-r--r-- 1 root root  14K Dec  6 16:36 EXAMPLES.md
drwxr-xr-x 2 root root 4.0K Dec  6 16:39 hack
-rw-r--r-- 1 root root  12K Dec  6 16:36 LICENSE
-rw-r--r-- 1 root root 1.1K Dec  6 16:36 Makefile
-rw-r--r-- 1 root root 6.6K Dec  6 16:36 README.md
-rw-r--r-- 1 root root 2.2K Dec  6 16:36 TESTING.md
drwxr-xr-x 5 root root 4.0K Dec  6 16:39 traefik
root@master1:~/traefik/traefik-helm-chart# ls -lh traefik/
total 336K
-rw-r--r-- 1 root root 257K Dec  6 16:36 Changelog.md
-rw-r--r-- 1 root root 1.7K Dec  6 16:36 Chart.yaml
drwxr-xr-x 2 root root 4.0K Dec  6 16:39 crds
-rw-r--r-- 1 root root 3.0K Dec  6 16:36 Guidelines.md
drwxr-xr-x 3 root root 4.0K Dec  6 16:39 templates
drwxr-xr-x 3 root root 4.0K Dec  6 16:39 tests
-rw-r--r-- 1 root root  17K Dec  6 16:36 VALUES.md
-rw-r--r-- 1 root root  34K Dec  6 16:36 values.yaml
```



## 3 Edit Config

```bash
cp traefik/values.yaml{,.bak}

vim traefik/values.yaml ## 关键的配置见下面内容
`
ingressRoute:
  dashboard:
    enabled: true

ingressClass:
  enabled: true
  isDefaultClass: false

providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true
    allowExternalNameServices: true


  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true

logs:
  general:
    level: DEBUG
  access:
    enabled: true

service:
  enabled: true
  single: true
  type: ClusterIP

additionalArguments:
  - "--log.level=DEBUG"

`
```



## 4 Install traefik

```bash
# 部署
root@master1:~/traefik/traefik-helm-chart# pwd 
/root/traefik/traefik-helm-chart

# docker pull docker.io/traefik:v2.10.5
# docker pull harbor.dockerregistry.com/kubernetes/traefik:v2.10.5 && docker tag harbor.dockerregistry.com/kubernetes/traefik:v2.10.5 docker.io/traefik:v2.10.5 

# helm install traefik --namespace=traefik-v2  traefik/traefik --version 24.0.0 --values=./traefik/values.yaml
helm install traefik --namespace=traefik-v2 --create-namespace ./traefik --values=./traefik/values.yaml

# 卸载（若需要）
helm uninstall traefik -n traefik-v2

```



## 5 Update traefik

```bash
# 更新
helm upgrade traefik --namespace=traefik-v2  ./traefik --values=./traefik/values.yaml 

```



## 6 View ingressclass

```bash
# 查看路由类
root@master1:~/traefik/traefik-helm-chart# kubectl get ingressclass
NAME      CONTROLLER                      PARAMETERS   AGE
traefik   traefik.io/ingress-controller   <none>       7m33s

```



## 7 Create ca

```bash
root@master1:~# cd ~/traefik/
root@master1:~/traefik# mkdir -p tls
root@master1:~/traefik# cd tls/

# rsa算法，创建证书
# 创建证书配置

tee openssl-ca.cnf <<-EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
default_days = 3650
default_md = sha256
prompt = no

[req_distinguished_name]
CN = My Root CA

[v3_ca]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical, CA:true 
EOF

tee openssl-server.cnf <<-EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no
default_days = 3650
default_md = sha256

[req_distinguished_name]
CN = traefik.k8scluster.com

[v3_req]
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = traefik.k8scluster.com
EOF

# 制作证书
openssl genrsa -out rootCA.key 2048 # 创建CA的私钥
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.crt -config openssl-ca.cnf # 创建并自签名根证书(CA)

openssl genrsa -out server.key 2048 # 创建服务器的私钥
openssl req -new -key server.key -out server.csr -config openssl-server.cnf # 创建服务器证书签名请求(CSR)
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out server.crt -days 3650 -sha256 -extensions v3_req -extfile openssl-server.cnf # 使用CA签署服务器证书

```



## 8 Create secret

```bash
# 创建密钥
kubectl create secret tls traefik-ingress-dashboard --namespace traefik-v2 --key ./server.key --cert ./server.crt
# 查看确认
kubectl -n traefik-v2 get secret traefik-ingress-dashboard -o jsonpath="{.data.tls\.crt}" | base64 --decode | openssl x509 -inform pem -text -noout

```



## 9 Create basic-auth

```bash
apt install apache2-utils -y

# htpasswd 命令生成一个有效的 htpasswd 字符串
root@master1:~/traefik/tls# htpasswd -nb admin Vg8NP15x60xcz3dffRS51038Lh3iHD8K
admin:$apr1$4UM4pVNl$VnlS3XyhpHGIhc2KQ5XRc1

# 创建密钥
#!!! 注意: 这里必须使用单引号，否则里面特殊符号就有其他意义了，会导致生成的密钥异常

kubectl create secret generic basic-auth --namespace=traefik-v2 --from-literal=auth='admin:$apr1$4UM4pVNl$VnlS3XyhpHGIhc2KQ5XRc1'  

```



## 10 Create IngressRoute

```bash
root@master1:~/traefik/tls# cd ~/traefik/



tee traefik-websecure-dashboard.yaml <<-eof
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: websecure-dashboard
  namespace: traefik-v2
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(\`traefik.k8scluster.com\`) && (PathPrefix(\`/dashboard\`) || PathPrefix(\`/api\`))
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
      middlewares:
        - name: basic-auth
          namespace: traefik-v2
  tls:
    secretName: traefik-ingress-dashboard
eof

kubectl create -f traefik-websecure-dashboard.yaml
```



## 11 Create Middleware

```bash

tee traefik-middleware.yaml <<-eof
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: traefik-v2
spec:
  basicAuth:
    secret: basic-auth
eof

kubectl create -f traefik-middleware.yaml
```



## 12 Expose svc

```bash
# 暴露traefik 服务
kubectl patch svc traefik -n traefik-v2 -p '{"spec": {"type": "NodePort", "ports": [{"name":"web", "port": 80, "targetPort": "web", "nodePort": 30882}, {"name":"websecure", "port": 443, "targetPort": "websecure", "nodePort": 30883}]}}'

# 查看确认
root@master1:~/traefik# kubectl get service -n traefik-v2
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
traefik   NodePort   10.109.232.131   <none>        80:30882/TCP,443:30883/TCP   23m
```



## 13 Add proxy

```bash
# openssl req -x509 -newkey rsa:2048 -sha256 -nodes \
#     -keyout traefik.k8scluster.com.key \
#     -days 3650 \
#     -subj "/C=CN/ST=Guangdong/L=Guangzhou/O=My Organization/OU=My Unit/CN=traefik.k8scluster.com" \
#     -out traefik.k8scluster.com.crt \
#     -extensions SAN \
#     -config <(echo "[req]"; 
#               echo distinguished_name=req; 
#               echo "[SAN]"; 
#               echo subjectAltName=DNS:traefik.k8scluster.com)

# upload files.
root@harbor:/data/nginx/web# tree .
.
├── docker-compose.yaml
└── nginx
    ├── conf.d
    │   ├── ks-console.conf
    │   ├── minio.conf
    │   └── traefik.conf
    ├── nginx.conf
    └── ssl
        ├── ks-console
        │   ├── kubesphere.k8scluster.com.crt
        │   └── kubesphere.k8scluster.com.key
        ├── minio
        │   ├── minio.objectstorage.com.crt
        │   └── minio.objectstorage.com.key
        └── traefik
            ├── traefik.k8scluster.com.crt
            └── traefik.k8scluster.com.key


# start svc
docker-compose up -d 

```

