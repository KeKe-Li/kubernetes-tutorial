#### kubernetes集群手动更新版本

本文主要介绍如何用 kubeadm 创建的 Kubernetes 集群从 1.15.2 版本升级到 1.16.1 版本.

通常我们使用 kubeadm 搭建的集群来更新是非常方便的，但是由于我们这里版本跨度太大，最新的版本是1.18.0,但是我们不能直接从 1.15.2更新到 1.18.0，kubeadm 的更新是不支持跨多个主版本的，所以我们现在是1.15.2，只能更新到 1.16.1 版本了，然后再重1.16.1 更新到 1.17.1等,不过版本更新的方式方法基本上都是一样的，所以后面要更新的话也挺简单了，下面我们就先将集群更新到  1.15.2版本。

#### 更新集群

首先我们需要保留 kubeadm config 文件：
```bash
> kubeadm config view
api:
  advertiseAddress: 10.151.30.11
  bindPort: 6443
  controlPlaneEndpoint: ""
auditPolicy:
  logDir: /var/log/kubernetes/audit
  logMaxAge: 2
  path: ""
authorizationModes:
- Node
- RBAC
certificatesDir: /etc/kubernetes/pki
cloudProvider: ""
criSocket: /var/run/dockershim.sock
etcd:
  caFile: ""
  certFile: ""
  dataDir: /var/lib/etcd
  endpoints: null
  image: ""
  keyFile: ""
imageRepository: k8s.gcr.io
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    clientConnection:
      acceptContentTypes: ""
      burst: 10
      contentType: application/vnd.kubernetes.protobuf
      kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
      qps: 5
    clusterCIDR: 10.244.0.0/16
    configSyncPeriod: 15m0s
    conntrack:
      max: null
      maxPerCore: 32768
      min: 131072
      tcpCloseWaitTimeout: 1h0m0s
      tcpEstablishedTimeout: 24h0m0s
    enableProfiling: false
    healthzBindAddress: 0.0.0.0:10256
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: 14
      minSyncPeriod: 0s
      syncPeriod: 30s
    ipvs:
      minSyncPeriod: 0s
      scheduler: ""
      syncPeriod: 30s
    metricsBindAddress: 127.0.0.1:10249
    mode: ""
    nodePortAddresses: null
    oomScoreAdj: -999
    portRange: ""
    resourceContainer: /kube-proxy
    udpIdleTimeout: 250ms
kubeletConfiguration: {}
kubernetesVersion: v1.10.0
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
nodeName: keke-master
privilegedPods: false
token: ""
tokenGroups:
- system:bootstrappers:kubeadm:default-node-token
tokenTTL: 24h0m0s
tokenUsages:
- signing
- authentication
unifiedControlPlaneImage: ""
```

将上面的imageRepository值更改为：`gcr.azk8s.cn/google_containers`，然后保存内容到文件 kubeadm-config.yaml 中（当然如果你的集群可以获取到 grc.io 的镜像可以不用更改）.


然后更新 kubeadm:
```bash
> yum makecache fast && yum install -y kubeadm-1.15.2-0 kubectl-1.15.2-0 
> kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.2", GitCommit:"f6278300bebbb750328ac16ee6dd3aa71213468", GitTreeState:"clean", BuildDate:"2019-08-05T09:20:51Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

因为 kubeadm upgrade plan 命令执行过程中会去 dl.k8s.io 获取版本信息，这个地址是需要科学方法才能访问的，所以我们可以先将 kubeadm 更新到目标版本，然后就可以查看到目标版本升级的一些信息了。

执行 upgrade plan 命令查看是否可以升级：

```bash
> kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
I0518 18:50:12.844665    9676 feature_gate.go:230] feature gates: &{map[]}
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.15.2
[upgrade/versions] kubeadm version: v1.15.2
[upgrade/versions] WARNING: Couldn't fetch latest stable version from the internet: unable to get URL "https://dl.k8s.io/release/stable.txt": Get https://dl.k8s.io/release/stable.txt: dial tcp 35.201.71.162:443: i/o timeout
[upgrade/versions] WARNING: Falling back to current kubeadm version as latest stable version
[upgrade/versions] WARNING: Couldn't fetch latest version in the v1.10 series from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.10.txt": Get https://dl.k8s.io/release/stable-1.10.txt: dial tcp 35.201.71.162:443: i/o timeout

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     3 x v1.15.2   v1.15.2

Upgrade to the latest stable version:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.15.2   v1.15.2
Controller Manager   v1.15.2   v1.15.2
Scheduler            v1.15.2   v1.15.2
Kube Proxy           v1.15.2   v1.15.2
CoreDNS                        1.1.3
Kube DNS             v1.15.2
Etcd                 3.1.12    3.2.18

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.15.2
```

我们可以先使用 dry-run 命令查看升级信息：
```bash
> $ kubeadm upgrade apply v1.15.2 --config kubeadm-config.yaml --dry-run
```
注意要通过--config指定上面保存的配置文件，该配置文件信息包含了上一个版本的集群信息以及修改搞得镜像地址。

查看了上面的升级信息确认无误后就可以执行升级操作了：
```bash
> kubeadm upgrade apply v1.16.0 --config kubeadm-config.yaml
kubeadm upgrade apply v1.16.0  --config kubeadm-config.yaml
[preflight] Running pre-flight checks.
I0518 18:57:29.134722   12284 feature_gate.go:230] feature gates: &{map[]}
[upgrade] Making sure the cluster is healthy:
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration options from a file: kubeadm-config.yaml
I0518 18:57:29.179231   12284 feature_gate.go:230] feature gates: &{map[]}
[upgrade/apply] Respecting the --cri-socket flag that is set with higher priority than the config file.
[upgrade/version] You have chosen to change the cluster version to "v1.16.0"
[upgrade/versions] Cluster version: v1.15.2
[upgrade/versions] kubeadm version: v1.15.2 
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Will prepull images for components [kube-apiserver kube-controller-manager kube-scheduler etcd]
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.16.0"...
Static pod: kube-apiserver-ydzs-master hash: 3abd7df4382a9b60f60819f84de40e11
Static pod: kube-controller-manager-ydzs-master hash: 1a0f3ccde96238d31012390b61109573
Static pod: kube-scheduler-ydzs-master hash: 2acb197d598c4730e3f5b159b241a81b
```

等几分钟时间看下,如果看到如下信息就证明集群升级成功了：
```bash
......
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS


[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.16.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
由于上面我们已经更新过 kubectl 了，现在我们用 kubectl 来查看下版本信息：

```bash
> kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.16.0", GitCommit:"f6278300bebbb750328ac16ee6dd3aa7d3549568", GitTreeState:"clean", BuildDate:"2019-08-05T09:23:26Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.16.0", GitCommit:"4485c6f18cee9a5d3c3b4e523bd27972b1b53892", GitTreeState:"clean", BuildDate:"2019-07-18T09:09:21Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

可以看到现在 Server 端和 Client 端都已经是 v1.16.0 版本了，然后查看下 Pod 信息：
```bash
> kubectl get pods -n kube-system
NAME                                             READY     STATUS    RESTARTS   AGE
authproxy-oauth2-proxy-798cff85fc-pc8x5          1/1       Running   0          12d
cert-manager-796fb45d79-wcrfp                    1/1       Running   2          12d
coredns-7f6746b7f-2cs2x                          1/1       Running   0          5m
coredns-7f6746b7f-clphf                          1/1       Running   0          5m
etcd-ydzs-master                                 1/1       Running   0          10m
kube-apiserver-ydzs-master                       1/1       Running   0          7m
kube-controller-manager-ydzs-master              1/1       Running   0          7m
kube-flannel-ds-amd64-jxzq9                      1/1       Running   8          20d
kube-flannel-ds-amd64-r56r9                      1/1       Running   3          20d
kube-flannel-ds-amd64-xw9fx                      1/1       Running   2          20d
kube-proxy-gqvdg                                 1/1       Running   0          3m
kube-proxy-sn7xb                                 1/1       Running   0          3m
kube-proxy-vbrr7                                 1/1       Running   0          2m
kube-scheduler-ydzs-master                       1/1       Running   0          6m
nginx-ingress-controller-587b4c68bf-vsqgm        1/1       Running   2          31d
nginx-ingress-default-backend-64fd9fd685-lmxhw   1/1       Running   1          31d
```
#### 更新 kubelet

可以看到我们之前的 kube-dns 服务已经被 coredns 取代了，这是因为在 v1.16.0 版本后就默认使用 coredns 了，我们也可以访问下集群中的服务看是否有影响，然后查看下集群的 Node 信息：
```bash
> kubectl get nodes
NAME              STATUS   ROLES    AGE    VERSION
keke-001          Ready    master   100d   v1.15.2
keke-002          Ready    master   100d   v1.15.2
keke-003          Ready    master   100d   v1.15.2
keke-004          Ready    <none>   99d    v1.15.2
keke-005          Ready    <none>   99d    v1.15.2
keke-006          Ready    <none>   99d    v1.15.3
```
可以看到版本并没有更新，这是因为节点上的 kubelet 还没有更新的，我们可以通过 kubelet 查看下版本：

```bash
> kubelet --version
Kubernetes v1.15.2
```
这个时候我们去手动更新下 kubelet：
```bash
> yum install -y kubelet-1.16.0-0
# 安装完成后查看下版本
> kubelet --version
Kubernetes v1.16.0
# 然后重启 kubelet 服务
> systemctl daemon-reload
> systemctl restart kubelet
> kubectl get nodes
NAME              STATUS   ROLES    AGE    VERSION
keke-001          Ready    master   100d   v1.16.0
keke-002          Ready    master   100d   v1.16.0
keke-003          Ready    master   100d   v1.16.0
keke-004          Ready    <none>   99d    v1.16.0
keke-005          Ready    <none>   99d    v1.16.0
keke-006          Ready    <none>   99d    v1.16.0
```
这里需要注意下:

1. 如果节点上 swap 没有关掉重启 kubelet 服务会报错，所以最好是关掉 swap，执行命令：`swapoff -a`即可。
2. Kubernetes v1.16.0 版本的 kubelet 默认使用的pod-infra-container-image镜像名称为：`k8s.gcr.io/pause:3.1`，所以最好先提前查看下集群节点上是否有这个镜像，因为我们之前 v1.15.0 版本的集群默认的名字为`k8s.gcr.io/pause-amd64:3.1`，所以如果节点上还是之前的 pause 镜像的话，需要先重新打下镜像 tag：
```bash
> docker tag k8s.gcr.io/pause-amd64:3.1 k8s.gcr.io/pause:3.1
```
没有的话可以提前下载到节点上也可以通过配置参数进行指定，在文件`/var/lib/kubelet/kubeadm-flags.env`中添加如下参数信息：
```bash
> KUBELET_KUBEADM_ARGS=--cgroup-driver=cgroupfs --cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d --network-plugin=cni --pod-infra-container-image=cnych/pause-amd64:3.1
```
可以看到我们更新了 kubelet 的节点版本信息已经更新了，同样的方式去把另外5个节点 kubelet 更新即可。

另外需要注意的是最好在节点上的 kubelet 更新之前将节点设置为不可调度，更新完成后再设置回来，可以避免不必要的错误。

最后看下升级后的集群：
```bash
> kubectl get nodes
NAME              STATUS   ROLES    AGE    VERSION
keke-001          Ready    master   100d   v1.16.0
keke-002          Ready    master   100d   v1.16.0
keke-003          Ready    master   100d   v1.16.0
keke-004          Ready    <none>   99d    v1.16.0
keke-005          Ready    <none>   99d    v1.16.0
keke-006          Ready    <none>   99d    v1.16.0
```
到这里我们的集群就升级成功了，我们可以用同样的方法将集群升级到 v1.17.x、v1.18.0 版本，而且升级过程中是不会影响到现有业务的。

更新集群后遇到一个问题是，其中一个节点上的 Pod 的 IP 没有使用 cni0 网桥的网段，而是使用的 docker0 的网段，比较奇怪，我们使用的 CNI 模式，而且查看了 Flannel 的相关配置参数都没有问题的，重新将虚拟网络设备重置了，然后重启该节点的 Pod 后:
```bash
> ifconfig cni0 down
> ip link delete cni0
> ifconfig flannel.1 down
> ip link delete flannel.1
> rm -rf /var/lib/cni/
```
所以最后是重新使用 kubeadm reset 了，重新加入集群才解决这个问题：

```bash
> kubeadm reset
# 使用下面命令获取加入节点的命令
> kubeadm token create --print-join-command
I0523 23:28:11.496418   18560 feature_gate.go:230] feature gates: &{map[]}
kubeadm join 10.151.30.11:6443 --token r7ilky.ppyy90a4ernkq7pj --discovery-token-ca-cert-hash sha256:e605d68721e7cf
```
重新执行上面的 join 命令，重新加入节点后恢复正常。
