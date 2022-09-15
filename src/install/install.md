#### kubernetes install 
```bash
> kubectl 

Basic Commands (Beginner):
  create         Create a resource from a file or from stdin.
  expose         使用 replication controller, service, deployment 或者 pod 并暴露它作为一个 新的
Kubernetes Service
  run            在集群中运行一个指定的镜像
  set            为 objects 设置一个指定的特征
 
Basic Commands (Intermediate):
  explain        查看资源的文档
  get            显示一个或更多 resources
  edit           在服务器上编辑一个资源
  delete         Delete resources by filenames, stdin, resources and names, or by resources and label selector
 
Deploy Commands:
  rollout        Manage the rollout of a resource
  scale          为 Deployment, ReplicaSet, Replication Controller 或者 Job 设置一个新的副本数量
  autoscale      自动调整一个 Deployment, ReplicaSet, 或者 ReplicationController 的副本数量
 
Cluster Management Commands:
  certificate    修改 certificate 资源.
  cluster-info   显示集群信息
  top            Display Resource (CPU/Memory/Storage) usage.
  cordon         标记 node 为 unschedulable
  uncordon       标记 node 为 schedulable
  drain          Drain node in preparation for maintenance
  taint          更新一个或者多个 node 上的 taints
 
Troubleshooting and Debugging Commands:
  describe       显示一个指定 resource 或者 group 的 resources 详情
  logs           输出容器在 pod 中的日志
  attach         Attach 到一个运行中的 container
  exec           在一个 container 中执行一个命令
  port-forward   Forward one or more local ports to a pod
  proxy          运行一个 proxy 到 Kubernetes API server
  cp             复制 files 和 directories 到 containers 和从容器中复制 files 和 directories.
  auth           Inspect authorization
 
Advanced Commands:
  apply          通过文件名或标准输入流(stdin)对资源进行配置
  patch          使用 strategic merge patch 更新一个资源的 field(s)
  replace        通过 filename 或者 stdin替换一个资源
  wait           Experimental: Wait for one condition on one or many resources
  convert        在不同的 API versions 转换配置文件
 
Settings Commands:
  label          更新在这个资源上的 labels
  annotate       更新一个资源的注解
  completion     Output shell completion code for the specified shell (bash or zsh)
 
Other Commands:
  alpha          Commands for features in alpha
  api-resources  Print the supported API resources on the server
  api-versions   Print the supported API versions on the server, in the form of "group/version"
  config         修改 kubeconfig 文件
  plugin         Runs a command-line plugin
  version        输出 client 和 server 的版本信息
```

### kubernetes 常用命令

1. 开启dashboard服务
```bash
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
2. Kubernetes可用的apiVersion版本
```bash
> kubectl api-versions

admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```
3. 手动设置Kubectl 上下文和配置
```bash

> kubectl config view # 显示合并的 kubeconfig 设置。
 
# 同时使用多个 kubeconfig 文件，并且查看合并的配置
> KUBECONFIG=~/.kube/config:~/.kube/kubconfig2 kubectl config view
 
# 查看名称为 “e2e” 的用户的密码
> kubectl config view -o jsonpath='{.users[?(@.name == "e2e")].user.password}'
 
> kubectl config current-context              # 显示当前上下文
> kubectl config use-context my-cluster-name  # 设置默认的上下文为 my-cluster-name
 
# 在 kubeconf 中添加一个支持基本鉴权的新集群。
> kubectl config set-credentials kubeuser/foo.kubernetes.com --username=kubeuser --password=kubepassword
 
# 使用特定的用户名和命名空间设置上下文。
> kubectl config set-context gce --user=cluster-admin --namespace=foo \
  && kubectl config use-context gce 
```

4. 执行kubectl命令，获取nodes的信息
```bash
> kubectl get nodes
NAME              STATUS   ROLES    AGE     VERSION
beta-101          Ready    master   4d10h   v1.15.2
beta-es-102       Ready    master   4d10h   v1.15.2
beta-es-103       Ready    master   4d9h    v1.15.2
es-101            Ready    <none>   4d9h    v1.15.2
es-102            Ready    <none>   4d9h    v1.15.2
```

5. kubectl create命令
```bash
> kubectl create -f nginx-deployment.yaml
```

nginx部署的YAML配置文件：
```markdown
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      imagePullSecrets:
        - name: dc-hspfd
      containers:
      # 应用的镜像
      - image: nginx
        name: nginx
        imagePullPolicy: IfNotPresent
        # 应用的内部端口
        ports:
        - containerPort: 80
          name: nginx80
        # 持久化挂接位置，在docker中      
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-data
        - mountPath: /etc/nginx
          name: nginx-conf
      volumes:
      # 宿主机上的目录
      - name: nginx-data
        nfs:
          path: /nfs/nginx
          server: 192.168.1.10
      - name: nginx-conf
        nfs:
          path: /k8s-nfs/nginx/conf
          server: 192.168.1.10
```

```bash
> kubectl create -f ./file.yaml           # 创建资源
> kubectl create -f ./data1.yaml -f ./data1.yaml     # 从多个文件创建资源
> kubectl create -f ./dir                        # 通过目录下的所有清单文件创建资源
> kubectl create -f https://git.io/vPieo         # 使用 url 获取清单创建资源
> kubectl run nginx --image=nginx                # 开启一个 nginx 实例
> kubectl explain pods,svc                       # 获取 pod 和服务清单的描述文档
 
# 通过标准输入创建多个 YAML 对象
> cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-less
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000"
EOF
 
# 使用多个 key 创建一个 secret
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: $(echo -n "s33msi4" | base64)
  username: $(echo -n "jane" | base64)
EOF
```

6. kubectl get 命令

通过此命令列出一个或多个资源对象，在这里通过kubectl get命令获取default命名空间下的所有部署。

```bash
> kubectl get deployment
```
```
# 具有基本输出的 get 命令
> kubectl get services                          # 列出命名空间下的所有 service
> kubectl get pods --all-namespaces             # 列出所有命名空间下的 pod
> kubectl get pods -o wide                      # 列出命名空间下所有 pod，带有更详细的信息
> kubectl get deployment my-dep                 # 列出特定的 deployment
> kubectl get pods --include-uninitialized      # 列出命名空间下所有的 pod，包括未初始化的对象
 
# 有详细输出的 describe 命令
> kubectl describe nodes my-node
> kubectl describe pods my-pod
 
> kubectl get services --sort-by=.metadata.name # List Services Sorted by Name
 
# 根据重启次数排序，列出所有 pod
> kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
 
# 查询带有标签 app=cassandra 的所有 pod，获取它们的 version 标签值
> kubectl get pods --selector=app=cassandra rc -o \
  jsonpath='{.items[*].metadata.labels.version}'
 
# 获取命名空间下所有运行中的 pod
> kubectl get pods --field-selector=status.phase=Running
 
# 所有所有节点的 ExternalIP
> kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
 
# 列出输出特定 RC 的所有 pod 的名称
# "jq" 命令对那些 jsonpath 看来太复杂的转换非常有用，可以在这找到：https://stedolan.github.io/jq/
> sel=${$(kubectl get rc my-rc --output=json | jq -j '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"')%?}
> echo $(kubectl get pods --selector=$sel --output=jsonpath={.items..metadata.name})
 
# 检查那些节点已经 ready
> JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"
 
# 列出某个 pod 目前在用的所有 Secret
> kubectl get pods -o json | jq '.items[].spec.containers[].env[]?.valueFrom.secretKeyRef.name' | grep -v null | sort | uniq
 
# 列出通过 timestamp 排序的所有 Event
```
7. kubectl describe命令
此命令用于显示一个或多个资源对象的详细信息，在这里通过获取上述nginx部署的信息。

```bash
> kubectl describe deployments/nginx
```

8. kubectl exec命令

此命令用于在Pod中的容器上执行一个命令，此处在nginx的一个容器上执行/bin/bash命令。

```bash
> kubectl exec -it nginx-c5cff9dcc-dr88t /bin/bash
```
9. kubectl logs命令
```bash
> kubectl logs nginx-c5cff9dcc-dr88t
```

10. kubectl delete命令

此命令用于删除集群中已存在的资源对象，可以通过指定名称、标签选择器、资源选择器等。

```bash
> kubectl delete -f ./pod.json                                              # 使用 pod.json 中指定的类型和名称删除 pod
> kubectl delete pod,service baz foo                                        # 删除名称为 "baz" 和 "foo" 的 pod 和 service
> kubectl delete pods,services -l name=myLabel                              # 删除带有标签 name=myLabel 的 pod 和 service
> kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除带有标签 name=myLabel 的 pod 和 service，包括未初始化的对象
> kubectl -n my-ns delete po,svc --all                                      # 删除命名空间 my-ns 下所有的 pod 和 service，包括未初始化的对象
```

11. kubectl rolling-update 命令

此命令用于滚动更新，对镜像、端口等的更新.

```bash
> kubectl rolling-update frontend-v1 -f frontend-v2.json           # 滚动更新 pod：frontend-v1
> kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2  # 变更资源的名称并更新镜像
> kubectl rolling-update frontend --image=image:v2                 # 更新 pod 的镜像
> kubectl rolling-update frontend-v1 frontend-v2 --rollback        # 中止进行中的过程
> cat pod.json | kubectl replace -f -                              # 根据传入标准输入的 JSON 替换一个 pod
 
# 强制替换，先删除，然后再重建资源。会导致服务中断。
> kubectl replace --force -f ./pod.json
 
# 为副本控制器（rc）创建服务，它开放 80 端口，并连接到容器的 8080 端口
> kubectl expose rc nginx --port=80 --target-port=8000
```    

13. kubectl patch命令
```bash
> kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}' # 部分更新节点
 
# 更新容器的镜像，spec.containers[*].name 是必需的，因为它们是一个合并键
> kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'
 
# 使用带有数组位置信息的 json 修补程序更新容器镜像
> kubectl patch pod valid-pod --type='json' -p='[{"op": "replace", "path": "/spec/containers/0/image", "value":"new image"}]'
 
# 使用带有数组位置信息的 json 修补程序禁用 deployment 的 livenessProbe
> kubectl patch deployment valid-deployment  --type json   -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/livenessProbe"}]'
 
# 增加新的元素到数组指定的位置中
> kubectl patch sa default --type='json' -p='[{"op": "add", "path": "/secrets/1", "value": {"name": "whatever" } }]'

```

14. kubectl edit命令
```bash
$ kubectl edit svc/docker-registry                      # 编辑名称为 docker-registry 的 service
$ KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用 alternative 编辑器
```

15. kubectl scale命令
```bash
> kubectl scale --replicas=3 rs/foo                                 # 缩放名称为 'foo' 的 replicaset，调整其副本数为 3
> kubectl scale --replicas=3 -f foo.yaml                            # 缩放在 "foo.yaml" 中指定的资源，调整其副本数为 3
> kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # 如果名称为 mysql 的 deployment 目前规模为 2，将其规模调整为 3
> kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # 缩放多个副本控制器
```

16. 与运行中的 pod 交互
```bash
> kubectl logs my-pod                                 # 转储 pod 日志到标准输出
> kubectl logs my-pod -c my-container                 # 有多个容器的情况下，转储 pod 中容器的日志到标准输出
> kubectl logs -f my-pod                              # pod 日志流向标准输出
> kubectl logs -f my-pod -c my-container              # 有多个容器的情况下，pod 中容器的日志流到标准输出
> kubectl run -i --tty busybox --image=busybox -- sh  # 使用交互的 shell 运行 pod
> kubectl attach my-pod -i                            # 关联到运行中的容器
> kubectl port-forward my-pod 5000:6000               # 在本地监听 5000 端口，然后转到 my-pod 的 6000 端口
> kubectl exec my-pod -- ls /                         # 1 个容器的情况下，在已经存在的 pod 中运行命令
> kubectl exec my-pod -c my-container -- ls /         # 多个容器的情况下，在已经存在的 pod 中运行命令
> kubectl top pod POD_NAME --containers               # 显示 pod 及其容器的度量
```    

17. 与 node 和集群交互
```bash
> kubectl cordon my-node                                                # 标记节点 my-node 为不可调度
> kubectl drain my-node                                                 # 准备维护时，排除节点 my-node
> kubectl uncordon my-node                                              # 标记节点 my-node 为可调度
> kubectl top node my-node                                              # 显示给定节点的度量值
> kubectl cluster-info                                                  # 显示 master 和 service 的地址
> kubectl cluster-info dump                                             # 将集群的当前状态转储到标准输出
> kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将集群的当前状态转储到目录 /path/to/cluster-state
 
# 如果带有该键和效果的污点已经存在，则将按指定的方式替换其值
> kubectl taint nodes foo dedicated=special-user:NoSchedule
```    

18. 启动dashboard

Kubernetes提供了以下四种访问服务的方式：

* kubectl proxy

kubectl proxy它在你的服务器与Kubernetes API之间创建一个代理，默认情况下，只能从本地访问（启动它的机器）。
```bash
> kubectl --namespace=kube-system get deployment kubernetes-dashboard
> sudo kubectl proxy                                              # 只能本地访问,
Starting to serve on 127.0.0.1:8001
```
我们也可以使用`--address`和`--accept-hosts`参数来允许外部访问：

```bash
> sudo kubectl proxy --address='0.0.0.0' --accept-hosts='^*$'     # 外部可访问
```    

然后我们在外网访问`http://<master-ip>:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`，可以成功访问到登录界面，但是却无法登录，这是因为Dashboard只允许`localhost`和`127.0.0.1`使用HTTP连接进行访问，而其它地址只允许使用HTTPS。
因此，如果需要在非本机访问Dashboard的话，只能选择其他访问方式。


* NodePort

NodePort是将节点直接暴露在外网的一种方式，只建议在开发环境，单节点的安装方式中使用。

启用NodePort很简单，只需执行`kubectl edit`命令进行编辑：
```bash
> sudo kubectl -n kube-system edit service kubernetes-dashboard
```
输出如下：
```bash
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kube-system"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: 2018-05-01T07:23:41Z
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "1750"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
  uid: 9329577a-4d10-11e8-a548-00155d000529
spec:
  clusterIP: 10.103.5.139
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```
然后我们将上面的 `type: ClusterIP` 修改为 `type: NodePort` ,保存后使用`kubectl get service`命令来查看自动生产的端口：
```bash
> kubectl -n kube-system get service kubernetes-dashboard
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4d11h
```

如上所示，`Dashboard`已经在31795端口上公开，现在可以在外部使用`https://<cluster-ip>:31795`进行访问。
需要注意的是，在多节点的集群中，必须找到运行`Dashboard`节点的IP来访问，而不是Master节点的IP，在本文的示例，我部署了两台服务器，MatserIP为192.168.10.8，ClusterIP为192.168.10.10。

遗憾的是，由于证书问题，我们无法访问，需要在部署Dashboard时指定有效的证书，才可以访问。
由于在正式环境中，并不推荐使用NodePort的方式来访问Dashboard，因此,关于如何为Dashboard配置证书可参考：(Certificate management)[https://github.com/kubernetes/dashboard/wiki/Certificate-management]。


* API Server

如果Kubernetes API服务器是公开的，并可以从外部访问，那我们可以直接使用API Server的方式来访问，也是比较推荐的方式。

Dashboard的访问地址为：
```markdown
https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```
但是返回的结果可能如下：
```bash
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "services \"https:kubernetes-dashboard:\" is forbidden: User \"system:anonymous\" cannot get services/proxy in the namespace \"kube-system\"",
  "reason": "Forbidden",
  "details": {
    "name": "https:kubernetes-dashboard:",
    "kind": "services"
  },
  "code": 403
}
```
这是因为最新版的k8s默认启用了RBAC，并为未认证用户赋予了一个默认的身份：`anonymous`。

对于API Server来说，它是使用证书进行认证的，我们需要先创建一个证书：

* 首先找到kubectl命令的配置文件，默认情况下为`/etc/kubernetes/admin.conf`，，我们已经复制到了`$HOME/.kube/config中`。

* 然后我们使用`client-certificate-data`和`client-key-data`生成一个p12文件，可使用下列命令：

```bash
# 生成client-certificate-data
> grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt

# 生成client-key-data
> grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key

# 生成p12
> openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

我们可以使用一开始创建的admin-user用户的token进行登录，一切OK。

注意:对于生产系统，我们应该为每个用户应该生成自己的证书，因为不同的用户会有不同的命名空间访问权限。
   
* Ingress

Ingress将开源的反向代理负载均衡器（如 Nginx、Apache、Haproxy等）与k8s进行集成，并可以动态的更新Nginx配置等，是比较灵活，更为推荐的暴露服务的方式.


19. 生成admin-user的登录令牌
```bash
> sudo kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```    
20. 查看service
```bash
> kubectl apply -f http://mirror.faasx.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```

```bash
> kubectl --namespace=kube-system get deployment kubernetes-dashboard
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1/1     1            1           31s
```

```bash
> kubectl --namespace=kube-system get service kubernetes-dashboard
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   ClusterIP   10.106.255.205   <none>        443/TCP   4m36s
```

21. 访问dashboard

如果Kubernetes API服务器是公开的，可以从外部访问的，就可以用API Server来访问。
访问的地址为:
```gotemplate
https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

22. 检查Kubernetes配置是否正确，集群是否可以访问
```bash
> sudo kubectl cluster-info
Kubernetes master is running at https://192.168.157.193:6443
KubeDNS is running at https://192.168.157.193:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

23. 列出使用的端口
```bash
> lsof -i :8080
```
24. kubectl proxy外界访问代理

如果不做kubectl proxy 则意味着外界访问api没任何限制,加上后可以做一些限制.
```bash
> kubectl proxy -h
```
25. 查询Linux暴露出来的端口
```bash
> netstat -tnlp
```

26. kubernetes的所有资源

kubernetes的所有资源都可以抽象为对象，分为几大类：

* 工作负载(workload)：Pod、ReplicaSet、Deployment、StatefulSet、DaemonSet、Job、Cronjob
* 服务发现和负载均衡：Service、Ingress
* 配置与存储：Volume、CSI、ConfigMap、Secret、DownwardAPI
* 集群级资源：Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding
* 元数据资源：HPA、PodTemplate、LimitRange


27. quest
```yaml
apiVersion: extensions/v1beta1                  # K8S对应的API版本
kind: Deployment                                # 对应的类型
metadata:
  name: test-go-service-deployment
  labels:
    name: test-go-service-deployment
  namespace: test                                # namespace
spec:
  replicas: 1                                   # 镜像副本数量
  template:
    metadata:
      labels:                                   # 容器的标签 可和service关联
        app: test-go-service
    spec:
      containers:
        - name: test-go-service             # 容器名和镜像
          image: registry-vpc.cn-hangzhou.aliyuncs.com/go-service:test
          imagePullPolicy: Always
          env:                                  # 环境变量
          - name: RUN_TIME
            value: test
          resources:                            # 资源限制
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
      imagePullSecrets:                         # 获取镜像需要的用户名密码
        - name: vpcregsecret
---
apiVersion: v1
kind: Service
metadata:
  namespace: test                                # 在哪个命名空间中创建
  name: test-go-service-service             # 名称
  labels:
    name: test-go-service-service
spec:
  type: ClusterIP                               # 开发端口的类型
  selector:                                     # service负载的容器需要有同样的labels
    app: test-go-service
  ports:
  - name: test-go-service-10001
    port: 10001                                    # 通过service来访问的端口
    targetPort: 10001                              # 对应容器的端口
```


28. 查看各组件信息
```bash
> kubectl get componentstatuses
```
29. 查看kubelet进程启动参数
```bash
> ps -ef| grep kubelet
```

30. 查看日志
```bash
> journalctl -u kubelet -f
```

31. 设为不可调度状态：
```bash
> kubectl cordon node1
```
32. 将pod赶到其他节点：
```bash
kubectl drain node1
```

33. 解除不可调度状态
```bash
kubectl uncordon node1
```






















