#### 如何创建Ingress资源
Ingress资源时基于HTTP虚拟主机或URL的转发规则，需要强调的是，这是一条转发规则。它在资源配置清单中的spec字段中嵌套了rules、backend和tls等字段进行定义。如下示例中定义了一个Ingress资源，其包含了一个转发规则：将发往myapp.magedu.com的请求，代理给一个名字为myapp的Service资源。
```yaml
apiVersion: extensions/v1beta1      
kind: Ingress       
metadata:           
  name: ingress-myapp   
  namespace: default     
  annotations:          
    kubernetes.io/ingress.class: "nginx"
spec:     
  rules:   
  - host: myapp.com   
    http:
      paths:       
      - path:       
        backend:    
          serviceName: myapp
          servicePort: 80
```

Ingress 中的spec字段是Ingress资源的核心组成部分，主要包含以下3个字段：

* rules：用于定义当前Ingress资源的转发规则列表；由rules定义规则，或没有匹配到规则时，所有的流量会转发到由backend定义的默认后端。
* backend：默认的后端用于服务那些没有匹配到任何规则的请求；定义Ingress资源时，必须要定义backend或rules两者之一，该字段用于让负载均衡器指定一个全局默认的后端。
* tls：TLS配置，目前仅支持通过默认端口443提供服务，如果要配置指定的列表成员指向不同的主机，则需要通过SNI TLS扩展机制来支持该功能。
* backend对象的定义由2个必要的字段组成：serviceName和servicePort，分别用于指定流量转发的后端目标Service资源名称和端口。
* rules对象由一系列的配置的Ingress资源的host规则组成，这些host规则用于将一个主机上的某个URL映射到相关后端Service对象，其定义格式如下：

```yaml
spec:
  rules:
  - hosts: <string>
    http:
      paths:
      - path:
        backend:
          serviceName: <string>
          servicePort: <string>
```
需要注意的是，.spec.rules.host属性值，目前暂不支持使用IP地址定义，也不支持IP:Port的格式，该字段留空，代表着通配所有主机名。
tls对象由2个内嵌的字段组成，仅在定义TLS主机的转发规则上使用。

hosts：包含于使用的TLS证书之内的主机名称字符串列表，因此，此处使用的主机名必须匹配tls Secret中的名称。
secretName： 用于引用SSL会话的secret对象名称，在基于SNI实现多主机路由的场景中， 此字段为可选.


#### Ingress资源类型

Ingress的资源类型有以下4种：

1. 单Service资源型Ingress
2. 基于URL路径进行流量转发
3. 基于主机名称的虚拟主机
4. TLS类型的Ingress资源

* 单Service资源型Ingress

暴露单个服务的方法有多种，如NodePort、LoadBanlancer等等，当然也可以使用Ingress来进行暴露单个服务，只需要为Ingress指定default backend即可，如下示例：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend:
    serviceName: my-svc
    servicePort: 80
```

Ingress控制器会为其分配一个IP地址接入请求流量，并将其转发至后端my-svc.


#### Ingress Nginx部署

使用Ingress功能步骤：
1. 安装部署ingress controller Pod
2. 部署后端服务
3. 部署ingress-nginx service
4. 部署ingress

从前面的描述我们知道，Ingress可以使用 yaml 的方式进行创建，从而得知Ingress也是标准的K8S资源，其定义的方式，也可以使用 explain 进行查看：
```bash
[root@k8s-master ~]# kubectl explain ingress
KIND:     Ingress
VERSION:  extensions/v1beta1

DESCRIPTION:
     Ingress is a collection of rules that allow inbound connections to reach
     the endpoints defined by a backend. An Ingress can be configured to give
     services externally-reachable urls, load balance traffic, terminate SSL,
     offer name based virtual hosting etc.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   spec <Object>
     Spec is the desired state of the Ingress. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

   status   <Object>
     Status is the current state of the Ingress. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status
```

1. 部署Ingress controller

* [ingress-nginx](https://github.com/kubernetes/ingress-nginx)

(1) 下载ingress相关的yaml:
```bash
[root@k8s-master ~]# mkdir ingress-nginx
[root@k8s-master ~]# cd ingress-nginx/
[root@k8s-master ingress-nginx]# for file in namespace.yaml configmap.yaml rbac.yaml tcp-services-configmap.yaml with-rbac.yaml udp-services-configmap.yaml default-backend.yaml;do wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/$file;done
[root@k8s-master ingress-nginx]# ll
total 28
-rw-r--r-- 1 root root  199 Sep 29 22:45 configmap.yaml #configmap用于为nginx从外部注入配置的
-rw-r--r-- 1 root root 1583 Sep 29 22:45 default-backend.yaml   #配置默认后端服务
-rw-r--r-- 1 root root   69 Sep 29 22:45 namespace.yaml #创建独立的名称空间
-rw-r--r-- 1 root root 2866 Sep 29 22:45 rbac.yaml  #rbac用于集群角色授权
-rw-r--r-- 1 root root  192 Sep 29 22:45 tcp-services-configmap.yaml
-rw-r--r-- 1 root root  192 Sep 29 22:45 udp-services-configmap.yaml
-rw-r--r-- 1 root root 2409 Sep 29 22:45 with-rbac.yaml
```

(2) 创建ingress-nginx名称空间
```bash
[root@k8s-master ingress-nginx]# cat namespace.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx

---
[root@k8s-master ingress-nginx]# kubectl apply -f namespace.yaml 
namespace/ingress-nginx created
```

(3) 创建ingress controller的pod
```bash
[root@k8s-master ingress-nginx]#  kubectl apply -f ./
configmap/nginx-configuration created
deployment.extensions/default-http-backend created
service/default-http-backend created
namespace/ingress-nginx configured
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
configmap/tcp-services created
configmap/udp-services created
deployment.extensions/nginx-ingress-controller created
[root@k8s-master ingress-nginx]# kubectl get pod -n ingress-nginx -w
NAME                                        READY     STATUS              RESTARTS   AGE
default-http-backend-7db7c45b69-gjrnl       0/1       ContainerCreating   0          35s
nginx-ingress-controller-6bd7c597cb-6pchv   0/1       ContainerCreating   0          34s
```

这里需要注意下,新版本的Kubernetes在安装部署中，需要从k8s.grc.io仓库中拉取所需镜像文件，但由于国内网络防火墙问题导致无法正常拉取。
docker.io仓库对google的容器做了镜像，可以通过下列命令下拉取相关镜像：
```bash
[root@k8s-node01 ~]# docker pull mirrorgooglecontainers/defaultbackend-amd64:1.5
1.5: Pulling from mirrorgooglecontainers/defaultbackend-amd64
9ecb1e82bb4a: Pull complete 
Digest: sha256:d08e129315e2dd093abfc16283cee19eabc18ae6b7cb8c2e26cc26888c6fc56a
Status: Downloaded newer image for mirrorgooglecontainers/defaultbackend-amd64:1.5

[root@k8s-node01 ~]# docker tag mirrorgooglecontainers/defaultbackend-amd64:1.5 k8s.gcr.io/defaultbackend-amd64:1.5
[root@k8s-node01 ~]# docker image ls
REPOSITORY                                    TAG                 IMAGE ID            CREATED             SIZE
mirrorgooglecontainers/defaultbackend-amd64   1.5                 b5af743e5984        34 hours ago        5.13MB
k8s.gcr.io/defaultbackend-amd64               1.5                 b5af743e5984        34 hours ago        5.13MB
```

2. 部署后端服务

(1) 查看ingress的配置清单选项

```bash
[root@k8s-master ingress-nginx]# kubectl explain ingress.spec
KIND:     Ingress
VERSION:  extensions/v1beta1

RESOURCE: spec <Object>

DESCRIPTION:
     Spec is the desired state of the Ingress. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

     IngressSpec describes the Ingress the user wishes to exist.

FIELDS:
   backend  <Object>     #定义后端有哪几个主机
     A default backend capable of servicing requests that don't match any rule.
     At least one of 'backend' or 'rules' must be specified. This field is
     optional to allow the loadbalancer controller or defaulting logic to
     specify a global default.

   rules    <[]Object>    #定义规则
     A list of host rules used to configure the Ingress. If unspecified, or no
     rule matches, all traffic is sent to the default backend.

   tls  <[]Object>
     TLS configuration. Currently the Ingress only supports a single TLS port,
     443. If multiple members of this list specify different hosts, they will be
     multiplexed on the same port according to the hostname specified through
     the SNI TLS extension, if the ingress controller fulfilling the ingress
     supports SNI.
```

(2) 部署后端服务
```bash
[root@k8s-master ingress-nginx]# cd ../mainfests/
[root@k8s-master mainfests]# mkdir ingress && cd ingress
[root@k8s-master ingress]# cp ../deploy-demo.yaml .
[root@k8s-master ingress]# vim deploy-demo.yaml 
#创建service为myapp
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  ports:
  - name: http
    targetPort: 80
    port: 80
---
#创建后端服务的pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-backend-pod
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      labels:
        app: myapp
        release: canary
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v2
        ports:
        - name: http
          containerPort: 80
[root@k8s-master ingress]# kubectl apply -f deploy-demo.yaml 
service/myapp created
deployment.apps/myapp-backend-pod unchanged
```

(3) 查看新建的后端服务pod
```bash
[root@k8s-master ingress]# kubectl get pods
NAME                                 READY     STATUS    RESTARTS   AGE
myapp-backend-pod-67f6f6b4dc-9jl9q   1/1       Running   0          7m
myapp-backend-pod-67f6f6b4dc-x5jsb   1/1       Running   0          7m
myapp-backend-pod-67f6f6b4dc-xzxbj   1/1       Running   0          7m
```

3. 部署ingress-nginx service

通过ingress-controller对外提供服务，现在还需要手动给ingress-controller建立一个service，接收集群外部流量。方法如下：
(1) 下载ingress-controller的yaml文件
```bash
[root@k8s-master ingress]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
[root@k8s-master ingress]# vim service-nodeport.yaml 
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30080
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
      nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
```
(2) 创建ingress-controller的service，并测试访问
```bash
[root@k8s-master ingress]# kubectl apply -f service-nodeport.yaml 
service/ingress-nginx created
[root@k8s-master ingress]# kubectl get svc -n ingress-nginx
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
default-http-backend   ClusterIP   10.104.41.201   <none>        80/TCP                       45m
ingress-nginx          NodePort    10.96.135.79    <none>        80:30080/TCP,443:30443/TCP   11s
```

此时访问:192.168.11.10:30080,此时这里应该是404 ，但是调度器是正常工作的，因为我们的后端服务还没有关联.


4. 部署ingress

(1) 编写ingress的配置清单
```bash
[root@k8s-master ingress]# vim ingress-myapp.yaml
apiVersion: extensions/v1beta1      #api版本
kind: Ingress       #清单类型
metadata:           #元数据
  name: ingress-myapp    #ingress的名称
  namespace: default     #所属名称空间
  annotations:           #注解信息
    kubernetes.io/ingress.class: "nginx"
spec:      #规格
  rules:   #定义后端转发的规则
  - host: myapp.magedu.com    #通过域名进行转发
    http:
      paths:       
      - path:       #配置访问路径，如果通过url进行转发，需要修改；空默认为访问的路径为"/"
        backend:    #配置后端服务
          serviceName: myapp
          servicePort: 80
[root@k8s-master ingress]# kubectl apply -f ingress-myapp.yaml
[root@k8s-master ingress]# kubectl get ingress
NAME            HOSTS              ADDRESS   PORTS     AGE
ingress-myapp   myapp.magedu.com             80        46s
```

(2) 查看ingress-myapp的详细信息
```bash
[root@k8s-master ingress]# kubectl describe ingress ingress-myapp
Name:             ingress-myapp
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host              Path  Backends
  ----              ----  --------
  myapp.magedu.com  
                       myapp:80 (<none>)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernetes.io/ingress.class":"nginx"},"name":"ingress-myapp","namespace":"default"},"spec":{"rules":[{"host":"myapp.magedu.com","http":{"paths":[{"backend":{"serviceName":"myapp","servicePort":80},"path":null}]}}]}}

  kubernetes.io/ingress.class:  nginx
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  1m    nginx-ingress-controller  Ingress default/ingress-myapp

[root@k8s-master ingress]# kubectl get pods -n ingress-nginx
NAME                                        READY     STATUS    RESTARTS   AGE
default-http-backend-7db7c45b69-fndwp       1/1       Running   0          31m
nginx-ingress-controller-6bd7c597cb-6pchv   1/1       Running   0          55m
```
(3) 进入nginx-ingress-controller进行查看是否注入了nginx的配置
```bash
[root@k8s-master ingress]# kubectl exec -n ingress-nginx -it nginx-ingress-controller-6bd7c597cb-6pchv -- /bin/bash
www-data@nginx-ingress-controller-6bd7c597cb-6pchv:/etc/nginx$ cat nginx.conf
......
    ## start server myapp.magedu.com
    server {
        server_name myapp.magedu.com ;
        
        listen 80;
        
        set $proxy_upstream_name "-";
        
        location / {
            
            set $namespace      "default";
            set $ingress_name   "ingress-myapp";
            set $service_name   "myapp";
            set $service_port   "80";
            set $location_path  "/";
            
            rewrite_by_lua_block {
                
                balancer.rewrite()
                
            }
            
            log_by_lua_block {
                
                balancer.log()
                
                monitor.call()
            }
......
```

(4) 修改本地host文件，进行访问
192.168.11.10 myapp.com
192.168.11.11 myapp.com


从前面的部署过程中，可以再次进行总结部署的流程如下：
* 下载Ingress-controller相关的YAML文件，并给Ingress-controller创建独立的名称空间；
* 部署后端的服务，如myapp，并通过service进行暴露；
* 部署Ingress-controller的service，以实现接入集群外部流量；
* 部署Ingress，进行定义规则，使Ingress-controller和后端服务的Pod组进行关联。


5. 构建TLS站点

(1) 准备证书
```bash
[root@k8s-master ingress]# openssl genrsa -out tls.key 2048 
Generating RSA private key, 2048 bit long modulus
.......+++
.......................+++
e is 65537 (0x10001)

[root@k8s-master ingress]# openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=tomcat.magedu.com
```

(2) 生成secret

```bash
[root@k8s-master ingress]# kubectl create secret tls tomcat-ingress-secret --cert=tls.crt --key=tls.key
secret/tomcat-ingress-secret created
[root@k8s-master ingress]# kubectl get secret
NAME                    TYPE                                  DATA      AGE
default-token-j5pf5     kubernetes.io/service-account-token   3         39d
tomcat-ingress-secret   kubernetes.io/tls                     2         9s
[root@k8s-master ingress]# kubectl describe secret tomcat-ingress-secret
Name:         tomcat-ingress-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1294 bytes
tls.key:  1679 bytes
```
(3) 创建ingress

```bash
[root@k8s-master ingress]# kubectl explain ingress.spec
[root@k8s-master ingress]# kubectl explain ingress.spec.tls
[root@k8s-master ingress]# cp ingress-tomcat.yaml ingress-tomcat-tls.yaml
[root@k8s-master ingress]# vim ingress-tomcat-tls.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-tomcat-tls
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - tomcat.magedu.com
    secretName: tomcat-ingress-secret
  rules:
  - host: tomcat.magedu.com
    http:
      paths:
      - path:
        backend:
          serviceName: tomcat
          servicePort: 8080

[root@k8s-master ingress]# kubectl apply -f ingress-tomcat-tls.yaml 
ingress.extensions/ingress-tomcat-tls created
[root@k8s-master ingress]# kubectl get ingress
NAME                 HOSTS               ADDRESS   PORTS     AGE
ingress-myapp        myapp.magedu.com              80        4h
ingress-tomcat-tls   tomcat.magedu.com             80, 443   5s
tomcat               tomcat.magedu.com             80        1h
[root@k8s-master ingress]# kubectl describe ingress ingress-tomcat-tls
Name:             ingress-tomcat-tls
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
TLS:
  tomcat-ingress-secret terminates tomcat.magedu.com
Rules:
  Host               Path  Backends
  ----               ----  --------
  tomcat.magedu.com  
                        tomcat:8080 (<none>)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernetes.io/ingress.class":"nginx"},"name":"ingress-tomcat-tls","namespace":"default"},"spec":{"rules":[{"host":"tomcat.magedu.com","http":{"paths":[{"backend":{"serviceName":"tomcat","servicePort":8080},"path":null}]}}],"tls":[{"hosts":["tomcat.magedu.com"],"secretName":"tomcat-ingress-secret"}]}}

  kubernetes.io/ingress.class:  nginx
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  20s   nginx-ingress-controller  Ingress default/ingress-tomcat-tls
```

