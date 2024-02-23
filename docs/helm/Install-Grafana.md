## 1 Download Helm Chart



```shell
mkdir -p grafana 
cd grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo grafana/grafana -l
helm search repo grafana/grafana --versions

#helm pull grafana/grafana --version 7.2.3 
tar xf grafana-7.2.3.tgz
cp -rvf grafana/values.yaml{,.bak}
```





## 2 Create Own Values.yaml

> 参考：https://github.com/grafana/helm-charts/blob/main/charts/grafana/README.md

```yaml
vim grafana/values.yaml

image:
  registry: docker.io
  repository: grafana/grafana
  tag: "10.2.3"  # 1 修改镜像标签为固定版本
  
  
  
ingress:
  enabled: true  
  hosts:
    - grafana.k8scluster.com 

ingress:
  enabled: true # 2 启用ingress
  annotations:  # 2.1 添加注释
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
  labels: {}
  path: /grafana/?(.*) # 2.2 添加访问后缀
  pathType: Prefix
  hosts:
    - grafana.k8scluster.com # 3 使用本地域名
    
persistence:
  type: pvc 
  enabled: true # 4 启用pvc
  storageClassName: default # 5 使用sc 名称
  accessModes:
    - ReadWriteOnce
  size: 1Gi  # 6 指定pvc 大小
  # annotations: {}
  

extraVolumeMounts:  # 7 指定挂载映射
  - name: grafana-data
    mountPath: /var/lib/grafana/plugins
    subPath: plugins
    readOnly: false
  - name: grafana-data
    mountPath: /var/lib/grafana/dashboards
    subPath: dashboards
    readOnly: false
  - name: grafana-data
    mountPath: /etc/grafana/provisioning
    subPath: provisioning
    readOnly: false

extraVolumes:
  - name: grafana-data
    persistentVolumeClaim:
      claimName: grafana-data-pvc   # 8 指定pvc 名称
```



## 3 Helm Install Grafana

```shell
# helm install grafana -f grafana/values.yaml ./grafana --namespace grafana --create-namespace

root@master2:~/grafana# helm install grafana -f grafana/values.yaml ./grafana --namespace grafana --create-namespace
NAME: grafana
LAST DEPLOYED: Tue Jan 23 17:00:55 2024
NAMESPACE: grafana
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.grafana.svc.cluster.local

   If you bind grafana to 80, please update values in values.yaml and reinstall:
     securityContext:
     runAsUser: 0
     runAsGroup: 0
     fsGroup: 0

   command:

   - "setcap"
   - "'cap_net_bind_service=+ep'"
   - "/usr/sbin/grafana-server &&"
   - "sh"
   - "/run.sh"
Details refer to https://grafana.com/docs/installation/configuration/#http-port.
   Or grafana would always crash.

   From outside the cluster, the server URL(s) are:
     http://grafana.k8scluster.com

3. Login with the password from step 1 and the username: admin
root@master2:~/grafana# kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
aL1Br7XMhp0DqTC8bNZ8eqwW15xnyR6YfAoTJyQn
```



## 4 Helm Update Grafana

```shell
# helm upgrade grafana -f grafana/values.yaml ./grafana --namespace grafana

root@master2:~/grafana# helm upgrade grafana -f grafana/values.yaml ./grafana --namespace grafana
W0123 17:10:35.217199 3106878 warnings.go:70] path /grafana/?(.*) cannot be used with pathType Prefix
Release "grafana" has been upgraded. Happy Helming!
NAME: grafana
LAST DEPLOYED: Tue Jan 23 17:10:33 2024
NAMESPACE: grafana
STATUS: deployed
REVISION: 2
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.grafana.svc.cluster.local

   If you bind grafana to 80, please update values in values.yaml and reinstall:
    securityContext:
     runAsUser: 0
     runAsGroup: 0
     fsGroup: 0

   command:

   - "setcap"
   - "'cap_net_bind_service=+ep'"
   - "/usr/sbin/grafana-server &&"
   - "sh"
   - "/run.sh"
   Details refer to https://grafana.com/docs/installation/configuration/#http-port.
   Or grafana would always crash.

   From outside the cluster, the server URL(s) are:
     http://grafana.k8scluster.com

3. Login with the password from step 1 and the username: admin
```

  



## 5 Login Dashboard

```shell
http://grafana.k8scluster.com:30886/grafana
```



## 6 Export Prometheus And Alertmanager

> 参考：https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/exposing-prometheus-and-alertmanager.md



1. create `ingress` 

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: prometheus-ingress
  namespace: kubesphere-monitoring-system
  annotations:
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
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-k8s
                port:
                  name: web
    - host: alertmanager.k8scluster.com
      http:
        paths:
          - path: /
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





## 7 Physical deployment



### 7.1 重置密码

[https://erdong.site/Grafana/grafana-admin-password-reset.html](https://erdong.site/Grafana/grafana-admin-password-reset.html)
[https://grafana.com/docs/grafana/latest/cli/#reset-admin-password](https://grafana.com/docs/grafana/latest/cli/#reset-admin-password)

```shell
grafana-cli admin reset-admin-password <new password>
```

### 7.2 rpm包编译构建

[https://github.com/grafana/grafana/issues/30963](https://github.com/grafana/grafana/issues/30963)

[https://github.com/goreleaser/nfpm](https://github.com/goreleaser/nfpm)

[https://goreleaser.com/install](https://goreleaser.com/install)

[https://nfpm.goreleaser.com/install/](https://nfpm.goreleaser.com/install/)

```shell
############################ 准备编译环境
# 0、安装常规工具
yum -y install git  
# 1、安装go环境，配置环境变量，不再演示，请自行百度
[root@demo BUILDSCRIPTS_CTREMEL]# cat /etc/profile|grep GO
export GOROOT=/usr/local/go
export GOPATH=/root/gocode
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
export GOPROXY=https://goproxy.io,direct
# 2、安装最新的nodejs，配置环境变量
# 安装最新的nodejs，过程略
# 安装cnpm,yarn,vue
npm root -g
npm install express -g
npm install -g cnpm --registry=https://registry.npm.taobao.org
npm install -g yarn
npm config set registry https://registry.npm.taobao.org
npm install -g @vue/cli
cnpm install -g @vue/cli
yarn -v
cnpm -v
# 配置环境变量
[root@demo BUILDSCRIPTS_CTREMEL]# tail -5 /etc/profile
#set nodejs npm yarn environment
export NODE_HOME=/usr/local/nodejs
export PATH=$NODE_HOME/bin:$PATH

#################################### 制作编译rpm包的配置文件
[root@demo ~]# mkdir -p gocode
[root@demo ~]# cd gocode/
[root@demo gocode]# ll
total 0
drwxr-xr-x 2 root root 58 Aug 17 09:52 bin
drwxr-xr-x 4 root root 30 Aug 16 16:38 BUILDSCRIPTS_CTREMEL
drwxr-xr-x 4 root root 30 Aug 16 16:57 pkg
drwxr-xr-x 5 root root 61 Aug 16 16:37 src
[root@demo gocode]# mkdir -p BUILDSCRIPTS_CTREMEL/{conf,dist}
[root@demo gocode]# ll BUILDSCRIPTS_CTREMEL/
total 0
drwxr-xr-x 2 root root 31 Aug 17 10:27 conf
drwxr-xr-x 2 root root 40 Aug 17 09:59 dist
[root@demo gocode]# cd BUILDSCRIPTS_CTREMEL/
[root@demo BUILDSCRIPTS_CTREMEL]# ll
total 0
drwxr-xr-x 2 root root 31 Aug 17 10:27 conf
drwxr-xr-x 2 root root 40 Aug 17 09:59 dist
[root@demo BUILDSCRIPTS_CTREMEL]#
[root@demo BUILDSCRIPTS_CTREMEL]# pwd
/root/gocode/BUILDSCRIPTS_CTREMEL
[root@demo BUILDSCRIPTS_CTREMEL]# vim conf/grafana_nfpm.yaml
[root@demo BUILDSCRIPTS_CTREMEL]# cat conf/grafana_nfpm.yaml
```

```
可能需要更改的4个地方：

1. version: "${GRAFANA_VERSION}"     // grafana版本

2. arch: "x86_64"    //CPU架构

3. release: 1 // rpm包的发行版本，第1次打rpm包

4. ./bin/linux-amd64   //当前编译打包的系统环境类型是linux-amd64
```

```yaml
name: "grafana"
arch: "x86_64"
platform: "linux"
version: "${GRAFANA_VERSION}"
section: "default"
priority: "extra"
release: 1
replaces:
- grafana
provides:
- grafana-server
- grafana-cli
depends:
- coreutils
- shadow-utils
maintainer: "<contact@grafana.com>"
description: |
  Grafana
vendor: "Grafana"
homepage: "https://grafana.com"
license: "Apache 2"
contents:
        - src: ./bin/linux-amd64/grafana-server
          dst: /usr/sbin/grafana-server
        - src: ./bin/linux-amd64/grafana-cli
          dst: /usr/sbin/grafana-cli
        - src: ./packaging/rpm/init.d/grafana-server
          dst: /etc/init.d/grafana-server
        - src: ./packaging/rpm/sysconfig/grafana-server
          dst: /etc/sysconfig/grafana-server
          type: config|noreplace
        - src: ./packaging/rpm/systemd/grafana-server.service
          dst: /usr/lib/systemd/system/grafana-server.service
        - src: ./public/
          dst: /usr/share/grafana/public
        - src: ./conf/
          dst: /usr/share/grafana/conf
empty_folders:
  - /etc/grafana
scripts:
    postinstall: ./packaging/rpm/control/postinst
rpm:
    scripts:
        posttrans: ./packaging/rpm/control/posttrans
```

```shell
############################################################# 开始编译
# 1、拉取官方代码并切换至v8.0.6的tag代码
mkdir -p ${GOPATH}/src/github.com/grafana
cd ${GOPATH}/src/github.com/grafana
# 这里原本是我本地gitlab上grafana版本的代码拉取
#rm -rf gd-grafana
#git clone -b dev git@10.1.1.1:dept-base/product/land-space/source3.1/yq-grafana.git
#cd yq-grafana
rm grafana  -rf 
git clone https://github.com/grafana/grafana.git
cd grafana
git checkout v8.0.6
git branch
# 2、删除本地grafana并移除配置、日志等
# 这么做是为了方便后面反复调试用，需要清理干净环境
yum -y remove grafana  >/dev/null 2>&1
systemctl stop grafana-server
sudo /bin/systemctl daemon-reload
rm -rf /usr/sbin/grafana-server /usr/sbin/grafana-cli /etc/init.d/grafana-server \
/usr/share/grafana /etc/grafana /var/lib/grafana /var/log/grafana /etc/sysconfig/grafana-server \
/usr/lib/systemd/system/grafana-server.service /usr/share/grafana/public /usr/share/grafana/conf
# 3、设置go代理
export GOPROXY=https://goproxy.io,direct
# 4、关闭安全git
git config --global http.sslVerify "false"
# 5、安装goreleaser、nfpm编译工具包
go install github.com/goreleaser/goreleaser@latest
go install github.com/goreleaser/nfpm/v2/cmd/nfpm@latest
# 6、开始初始化、构建后端工程
go run build.go setup
go run build.go build
# 7、下载前端依赖、构建前端
yarn install --pure-lockfile
yarn build
# 8、设置Grafana版本号
export GRAFANA_VERSION="v8.0.6"
# 9、使用提前改好的rpm打包配置文件+nfpm打包工具来制作rpm包
${GOPATH}/bin/nfpm pkg --packager rpm --config /root/gocode/BUILDSCRIPTS_CTREMEL/conf/grafana_nfpm.yaml --target /root/gocode/BUILDSCRIPTS_CTREMEL/dist
# 10、开始安装Grafana
yum -y install /root/gocode/BUILDSCRIPTS_CTREMEL/dist/grafana-0.0.0~rc0-1.x86_64.rpm
# 11、设置刷新系统服务、开机自启、启动
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable grafana-server.service
sudo /bin/systemctl start grafana-server.service
```


### 7.3 数据持久化

```shell
# 0
# 找一台现成的PG，建库建用户赋权
create user grafana with password 'grafana';
create database grafana owner=grafana encoding='UTF8';
grant all on database grafana to grafana;

# 1
# rpm/yum/脚本 重装grafana

# 2
# 替换配置文件
mkdir -p grafana_bakup
cp  -rvf /etc/grafana/grafana.ini grafana_bakup
# 修改ini文件中的配置
[database]
type = postgres         # 数据库类型
host = 10.0.1.1:5432  # 数据库地址、端口
name = grafana      # 数据库库名
user = grafana      # 数据库用户名
password = grafana  # 数据库密码
ssl_mode = disable  # 打开注释
path = grafana.db   # 打开注释 


rm /etc/grafana/grafana.ini
cp  -rvf grafana_bakup/grafana.ini /etc/grafana/grafana.ini

# 3
# 修改文件权限
chown root.grafana /etc/grafana/grafana.ini

# 4
# 重启服务
systemctl restart grafana-server.service

# 5 
# Web 界面操作
# 重新添加数据源
# 重新导入模板
# 这样后面更新部署就不再需要重新添加数据源和导入模板了
```


### 7.4 查看版本

```shell
rpm -qa grafana
grafana-server -v
```

### 7.5 升级版本

```shell
#升级
rpm -Uvh grafana-6.4.0-1.x86_64.rpm 
#重启服务
systemctl  restart grafana-server
```

### 7.6 Windows 服务注册

```cmd
nssm install prometheus C:\soft\prometheus-2.49.0-rc.2.windows-amd64\prometheus.exe
nssm set prometheus AppDirectory C:\soft\prometheus-2.49.0-rc.2.windows-amd64
nssm set prometheus AppParameters "--config.file=C:\soft\prometheus-2.49.0-rc.2.windows-amd64\prometheus.yml"
nssm set prometheus Start SERVICE_AUTO_START

nssm install grafana C:\soft\grafana-v10.2.3\bin\grafana-server.exe
nssm set grafana AppDirectory  C:\soft\grafana-v10.2.3
nssm set grafana Start SERVICE_AUTO_START

```

