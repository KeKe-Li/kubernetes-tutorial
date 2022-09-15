#### Drone 集成到Gitlab

在配置文件中，我们设置 docker-compose.yml 的格式为 2 号版本，定义两种服务。

drone-server ：使用 drone/drone:0.8.2 版本镜像，将启动监听 3800 上的主 Drone 服务容器，9000 端口来开放给 Agent。我们将在容器内挂载 /etc/drone 目录，以便 drone 可以保留数据。配置服务自动重新启动，并添加一些环境变量。
drone-agent：使用 drone/agent:0.8.2 版本镜像，将 docker 启动句柄挂载到容器 /var/run/docker.sock 文件中，以便 drone 可以使用 docker 来执行镜像构建任务。环境变量中需要配置 drone server 的地址以及 server 的密钥，以便于 server 进行通信。


Drone 支持 GitLab，使用下面的环境变量来配置使用 GitLab。
```docker-compose
version: '2'

services:
  drone-server:
    image: drone/drone:0.7
    ports:
     - 3800:8000
     - 9000
    volumes:
      - /etc/drone:/var/lib/drone/
    restart: always
    environment:
      - DRONE_OPEN=true
      - DRONE_GITLAB=true
      - DRONE_GITLAB_CLIENT={DRONE_GITLAB_CLIENT}
      - DRONE_GITLAB_SECRET={DRONE_GITLAB_SECRET}
      - DRONE_GITLAB_URL=http://gitlab.test.com
      - DRONE_SECRET=${DRONE_SECRET}
  drone-agent:
    image: drone/drone:0.7
    command: agent
    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_SERVER=ws://drone-server:8000/ws/broker
      - DRONE_SECRET=${DRONE_SECRET}
```
其中的通信密钥相当于 drone 的密码，最好为一个长传的随机字串防止被破解。可以在命令行中使用一下内容生成：
```markdown
 > LC_ALL=C </dev/urandom tr -dc A-Za-z0-9 | head -c 65 && echo
```


#### 配置
下面是所有的配置选项。一般来说，使用默认配置可以满足绝大部分的安装需求：
````markdown

* DRONE_GITLAB=true
  true 使用 GitLab
* DRONE_ADMIN=drone
  drone注册后的管理员用户  
* DRONE_GITLAB_URL=https://gitlab.com
  GitLab Server 地址
* DRONE_GITLAB_CLIENT
  GitLab oauth2 client id
* DRONE_GITLAB_SECRET
  GitLab oauth2 client secret
* DRONE_GITLAB_GIT_USERNAME
  可选，使用单一用户来克隆所有仓库，这个用户的用户名
* DRONE_GITLAB_GIT_PASSWORD
  可选，使用单一用户来克隆所有仓库，这个用户的密码
* DRONE_GITLAB_SKIP_VERIFY=false
  设置 true 来取消 SSL 检查
* DRONE_GITLAB_PRIVATE_MODE=false
  如果 GitLab 以 private 私有模式运行，应设置为 true

````

#### 注册应用程序
在 GitLab 上注册一个应用，并生成一个客户端和密钥。访问账户设置（account settings）页面，选择 Applications 页面，点击 New Application 。

使用下面的认证回调 URL（Authorization callback URL），请修改域名为自定义域名： http://drone.mycompany.com/authorize


#### 启动服务

配置文件完成之后，我们就可以使用以下命令启动服务了：

```bash
 > docker-compose  up -d
```

docker-compose 会自动帮我们去下载镜像并根据配置初始化容器。一切就绪之后，我们使用 <host>:3800 就可以访问到 drone 了。

#### 配置 Nginx

我们配置下 Nginx 做下反向代理就可以了，以下是具体的 nginx 配置文件示例：
```markdown
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
server {
    listen 80;
    server_name ci.eming.li;
    set $drone_port 3800;
    location / {
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:$drone_port;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_buffering off;
        chunked_transfer_encoding off;
    }
    location ~* /ws {
        proxy_pass http://127.0.0.1:$drone_port;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
    }
}

```
修改 server_name 和 $drone_port 为正确的值即可，最后记得别忘了重启 Nginx 服务。
这样我们就能直接使用 http://gitlab.test.com来访问 drone 了。


#### 进程守护
刚才使用 docker-compose up 命令启动如果退出终端了之后服务就会停止，所以我们需要后台执行。我们可以直接使用 nohup 方法启动：

```bash
 > nohup docker-compose -f docker-compose.yml up &
```
除了使用 nohup 之外，我们还可以使用 systemctl 来启动进程。我们创建一个 drone.service 服务文件：

```bash
 > vim /etc/systemd/system/drone.service
 
 [Unit]
 Description=Drone server
 After=docker.service nginx.service
 [Service]
 Restart=always
 ExecStart=/usr/local/bin/docker-compose -f /etc/drone/docker-compose.yml up
 ExecStop=/usr/local/bin/docker-compose -f /etc/drone/docker-compose.yml stop
 [Install]
 WantedBy=multi-user.target
```

第一部分告诉 systemd 在 Docker 和 Nginx 可用之后启动此服务。
 第二部分告诉 init 系统在发生故障时自动重新启动服务。 
 然后，它使用 Docker Compose 和我们之前创建的配置文件定义启动和停止 Drone 服务的命令。 
最后，最后一节定义了如何使服务在启动时启动。完成后保存文件并使用如下命令启动服务：

```bash
 > systemctl start drone
```
使用如下命令可以查看服务启动状态：  

```bash
 > systemctl status drone
```

#### Drone CI 配置文件
Drone CI 对一个项目进行 CI 构建取决于两个因素，第一必须保证该项目在 Drone 控制面板中开启了构建(构建按钮开启)，第二保证项目根目录下存在 .drone.yml；满足这两点后每次提交 Drone 就会根据 .drone.yml 中配置进行按步骤构建；本示例中 .drone.yml 配置如下

```yaml
clone:
  git:
    image: plugins/git

pipeline:

  backend:
    image: reg.mritd.me/base/build:2.1.5
    commands:
      - gradle --no-daemon clean assemble
    when:
      branch:
        event: [ push, pull_request ]
        include: [ master ]
        exclude: [ develop ]

#  rebuild-cache:
#    image: drillster/drone-volume-cache
#    rebuild: true
#    mount:
#      - ./build
#    volumes:
#      - /data/drone/$DRONE_COMMIT_SHA:/cache

  docker:
    image: mritd/docker-kubectl:v1.8.8
    commands:
      - bash build_image.sh
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock



# Pipeline Conditions
branches:
  include: [ master, feature/* ]
  exclude: [ develop, test/* ]
```

Drone CI 配置文件为 docker compose 的超集，Drone CI 构建思想是使用不同的阶段定义完成对 CI 流程的整体划分，然后每个阶段内定义不同的任务(task)，
这些任务所有操作无论是 build、package 等全部由单独的 Docker 镜像完成，同时以 plugins 开头的 image 被解释为内部插件；其他的插件实际上可以看做为标准的 Docker image


第一段 clone 配置声明了源码版本控制系统拉取方式，具体参见 cloning部分，定义后 Drone CI 将自动拉取源码

此后的 pipeline 配置段为定义整个 CI 流程段，该段中可以自定义具体 task，比如后端构建可以取名字为 backend，前端构建可以叫做 frontend；中间可以穿插辅助的如打包 docker 镜像等 task；同 GitLab CI 一样，
Agent 在使用 Docker 进行构建时必然涉及到拉取私有镜像，Drone CI 想要拉取私有镜像目前仅能通过 cli 命令行进行设置，而且仅针对项目级设置(全局需要企业版…这也行)
 
 ```bash
 drone registry add --repository drone/DroneCI-TestProject --hostname reg.mritd.me --username gitlab --password 123456

 ``` 
在构建时需要注意一点，Drone CI 不同的 task 之间共享源码文件，也就是说如果你在第一个task中对源码或者编译后的发布物做了什么更改,
在下一个 task 中同样可见，Drone CI 并没有 GitLab CI 在每个 task 中都进行还原的机制
  
除此之外，某些特殊性的挂载行为默认也是不被允许的，需要在 Drone CI 中对项目做 Trusted 设置.  
  
  
  
#### 问题

至此 drone 的安装就结束了。不过我在具体使用的过程中有两个属于安装的小问题，在此记录一下：  

1. 容器无法访问网络

一台服务器上安装，访问后发现无法连接 Github 内部的所有网络服务都不正常。 

docker nat 的问题.

2. 构建任务无故退出

实际使用过程中我发现我的构建任务总是莫名其妙就失败，提示我被 killed 掉，并显示 exit code 137。
最开始我一直在查 137 退出码，发现就是 kill -9 的退出，不明所以。
后来偶然尝试以 docker killed 为关键词搜索，才发现原来是内存爆了 docker 执行 OOM 把容器杀掉了。

#### 总结

这样我们的安装过程就结束了，访问 drone，使用对应仓库的账户登录。如果是 Github 的话会使用 OAuth 连接到 Github 进行授权申请
