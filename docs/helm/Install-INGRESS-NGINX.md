## 1 pull package

```shell
# https://kubernetes.github.io/ingress-nginx/deploy/
# https://blog.thecloudside.com/deploying-public-private-nginx-ingress-controllers-with-http-s-loadbalancer-in-gke-dcf894197fb7

root@master1:~# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
root@master1:~# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "projectcalico" chart repository
...Successfully got an update from the "gitlab" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈Happy Helming!⎈

root@master1:~# helm search repo ingress-nginx/ingress-nginx -l
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
ingress-nginx/ingress-nginx     4.9.0           1.9.5           Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     4.8.3           1.9.4           Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     4.8.2           1.9.3           Ingress controller for Kubernetes using NGINX a...
...
ingress-nginx/ingress-nginx     2.0.1           0.31.1          Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     2.0.0           0.31.0          An nginx Ingress controller that uses ConfigMap...
root@master1:~# 
root@master1:~# helm pull ingress-nginx/ingress-nginx --version 4.8.3
root@master1:~# mkdir -p ingress-nginx 
root@master1:~# cd ingress-nginx
root@master1:~/ingress-nginx# mv ../ingress-nginx-4.8.3.tgz .
root@master1:~/ingress-nginx# ll
total 48
-rw-r--r-- 1 root root 47117 Dec 23 10:00 ingress-nginx-4.8.3.tgz
root@master1:~/ingress-nginx# tar xf ingress-nginx-4.8.3.tgz 
root@master1:~/ingress-nginx# ll 
total 52
drwxr-xr-x 5 root root  4096 Dec 23 10:02 ingress-nginx
-rw-r--r-- 1 root root 47117 Dec 23 10:00 ingress-nginx-4.8.3.tgz
root@master1:~/ingress-nginx# cp ingress-nginx/values.yaml{,.bak}

```



## 2 change values.yaml

```yaml
root@master1:~/ingress-nginx# vim ingress-nginx/values.yaml # 暴露NodePort

  service:
    enabled: true
    #type: LoadBalancer
    ### type: NodePort
    ### nodePorts:
    ###   http: 32080
    ###   https: 32443
    ###   tcp:
    ###     8080: 32808
    #nodePorts:
    #  http: ""
    #  https: ""
    #  tcp: {}
    #  udp: {}
    type: NodePort  
    nodePorts:
      http: "30886"  
      https: "30887"  
      tcp: {}
      udp: {}
```





## 3 install ingress

```shell
# helm upgrade --install ingress-nginx ingress-nginx \
#   --repo https://kubernetes.github.io/ingress-nginx \
#   --namespace ingress-nginx --create-namespace

root@master1:~/ingress-nginx# helm upgrade --install ingress-nginx ./ingress-nginx   --namespace ingress-nginx --create-namespace

Release "ingress-nginx" does not exist. Installing it now.
NAME: ingress-nginx
LAST DEPLOYED: Sat Dec 23 11:00:59 2023
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
Get the application URL by running these commands:
  export HTTP_NODE_PORT=30886
  export HTTPS_NODE_PORT=30887
  export NODE_IP=$(kubectl --namespace ingress-nginx get nodes -o jsonpath="{.items[0].status.addresses[1].address}")

  echo "Visit http://$NODE_IP:$HTTP_NODE_PORT to access your application via HTTP."
  echo "Visit https://$NODE_IP:$HTTPS_NODE_PORT to access your application via HTTPS."

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls

```





## 4 test ingress

```yaml
root@master1:~/ingress-nginx# kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo
deployment.apps/demo created
service/demo exposed
root@master1:~/ingress-nginx# kubectl create ingress demo-localhost --class=nginx \
  --rule="demo.localdev.me/*=demo:80"
ingress.networking.k8s.io/demo-localhost created
root@master1:~/ingress-nginx# kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

Handling connection for 8080
Handling connection for 8080


root@master1:~/ingress-nginx# 
root@master1:~/ingress-nginx# curl --resolve demo.localdev.me:8080:127.0.0.1 http://demo.localdev.me:8080
<html><body><h1>It works!</h1></body></html>



root@master1:~/ingress-nginx# kubectl get service ingress-nginx-controller --namespace=ingress-nginx
NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.100.61.186   <none>        80:30886/TCP,443:30887/TCP   13m
root@master1:~/ingress-nginx# kubectl create ingress demo --class=nginx \
  --rule="www.demo.io/*=demo:80"
ingress.networking.k8s.io/demo created

# 添加本地hosts 
192.168.1.120 www.demo.io 

# 浏览器访问
http://www.demo.io:30886/
```