#### Drone 使用

.drone.yml
````yaml
workspace:
  base: /go
  # 指定git clone到的地方, 应该放在gopath下, 才能正常编译
  path: src/spike

pipeline:
  build-develop:
    image: golang:1.9
    commands:
      - pwd
      - go version
      # 如果要在alpine上运行编译后文件则必须添加这些参数, 参见http://docs.drone.io/creating-custom-plugins-golang/
      - GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o app
      - cp -f config.test.yml config.yml
    when:
      branch: develop

  # 需要在drone填写secrets: docker_username, docker_password
  publish-develop:
    image: plugins/docker
    mirror: https://docker.mirrors.ustc.edu.cn
    registry: registry-internal.cn-hangzhou.aliyuncs.com # 仓库
    repo: registry-internal.cn-hangzhou.aliyuncs.com/zhuzi/drone-test # docker仓库地址
    secrets: [ docker_username, docker_password ]
    tags:
      - test
    when:
      branch: develop

  # 插件会干两件事情, 1. upgrade 2. finish upgrade
  # 注意如果这个服务不在Active状态就不能正常升级, 需要登录rancher修改状态, 正常情况下不会发生这个错误.
  rancher-develop:
    image: peloton/drone-rancher
    url: http://rancher.bysir.store/v1
    access_key: ""
    secret_key: ""
    service: app/drone-test
    # 为了使rancher能拉取到私有镜像, 需要在rancher控制面板"基础架构->镜像库"添加这个私有镜像库
    docker_image: registry-internal.cn-hangzhou.aliyuncs.com/zhuzi/drone-test:test
    start_first: true # 先启动新服务, 后停止原服务. 如果为false则先关闭原服务再启动
    confirm: true
    timeout: 100 # 如果rancher没在这个时间内升级成功则报错, 服务大小等差异会导致升级时间不一样, 可根据自己业务修改超时时间.
    when:
      branch: develop


```
这个pipeline主要是：
* build: 编译go项目为可执行文件
* publish: 将可执行文件通过Dockerfile打包并发布到仓库
* rancher: 调用rancher的api实现一个应用的升级. 使用到了一个第三方插件peloton/drone-rancher.


* workspace

workspace指定pipeline的工作目录, 我们会在build中pwd看到当前目录是/go/src/spike, 
为什么我们需要指定到/go目录下, 因为在golang:1.9的镜像中, gopath就是/go, 我们要go build当然要在gopath下执行.

* build

build步骤很简单只是go build.如果用go包管理glide把依赖vender提交了，那么就可以省去go get了.
docker在构建的时候都是以一个空白镜像golang:1.9作为基础的, 如果不提交vendor就需要每次构建都go get, 十分耗时.
当然还有办法就是提交一个已经按照好go包的基础镜像到registry里, 在build中的image就换成你提交的镜像. 相比之下更简单的方法就是提交vendor目录.

* publish

publish使用到了plugins/docker插件, 这个插件是drone写的, 用于发布docker镜像. 它的作用就是构建一个镜像, 并push到registry.

需要配置的值有:

1. registry: 仓库registry, 如hub.docker.com的registry地址是https://index.docker.io/v1/.

2. repo: 在docker仓库下的项目名称.

3. secrets: drone用于传递密钥的实现方式

在plugins/docker插件中, 构建项目镜像是通过Dockerfile来的, 所以我们还需要在项目根编写一个Dockerfile.

```dockerfile

FROM alpine:latest

COPY gokit_start /

WORKDIR /

ENTRYPOINT ["./gokit_start"]

```
ENTRYPOINT是容器启动后的运行入口, "./gokit_start"是示例项目build后的二进制文件.

* deply

发布流程就是通过SSH登陆上要部署程序的服务器pull下刚刚publish的镜像并启动.

登陆SSH就需要配置ssh_key或者ssh_password, 更多详情看appleboy/ssh这个插件的文档, 这里推荐使用ssh_key,
 我们需要在drone的secrets添加一项ssh_key值为私钥, 然后我们将与之匹配的公钥放在服务器上.ssh/authorized_keys里, 这样就能使用ssh_key登陆上服务器并执行script.

