
#### Dockerfile

```dockerfile
#母镜像
FROM golang:latest as build
#维护者信息
MAINTAINER keke

ENV GOPROXY https://goproxy.io/
# go module开启
ENV GO111MODULE on

WORKDIR /go/cache

# 添加go mod
ADD go.mod .
ADD go.sum .

# 构建缓存包含了该项所有的依赖起到加速构建的作用
RUN go mod download

#工作目录
WORKDIR /go/release

#将文件复制到镜像中
ADD . .

# ldflags中-s: 省略符号表和调试信息,-w: 省略DWARF符号表
RUN GOOS=linux CGO_ENABLED=0 go build -ldflags="-s -w" -installsuffix cgo -o quest main.go

# scratch空的基础镜像，最小的基础镜像
# busybox带一些常用的工具，方便调试， 以及它的一些扩展busybox:glibc
# alpine另一个常用的基础镜像，带包管理功能，方便下载其它依赖的包
FROM scratch as prod

# 配置镜像的时间区
COPY --from=build /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 配置镜像的证书
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
# 将构建的可执行文件复制到新镜像中
COPY --from=build /go/release/quest /
# 将配置赋值到新镜像
COPY --from=build /go/release/conf/conf.yaml /

EXPOSE 10835

CMD ["/quest"]

```

```dockerfile
FROM golang:1.11.1-alpine3.8 as build-env
# All these steps will be cached
RUN mkdir /hello
WORKDIR /hello
COPY go.mod . # <- COPY go.mod and go.sum files to the workspace
COPY go.sum .

# Get dependancies - will also be cached if we won't change mod/sum
RUN go mod download
# COPY the source code as the last step
COPY . .

# Build the binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -installsuffix cgo -o /go/bin/hello
FROM scratch # <- Second step to build minimal image
COPY --from=build-env /go/bin/hello /go/bin/hello
ENTRYPOINT ["/go/bin/hello"]
```

```dockerfile
# Accept the Go version for the image to be set as a build argument.
# Default to Go 1.11
ARG GO_VERSION=1.11

# First stage: build the executable.
FROM golang:${GO_VERSION}-alpine AS builder

# Create the user and group files that will be used in the running container to
# run the process as an unprivileged user.
RUN mkdir /user && \
    echo 'nobody:x:65534:65534:nobody:/:' > /user/passwd && \
    echo 'nobody:x:65534:' > /user/group

# Install the Certificate-Authority certificates for the app to be able to make
# calls to HTTPS endpoints.
RUN apk add --no-cache ca-certificates

# Set the environment variables for the go command:
# * CGO_ENABLED=0 to build a statically-linked executable
# * GOFLAGS=-mod=vendor to force `go build` to look into the `/vendor` folder.
ENV CGO_ENABLED=0 GOFLAGS=-mod=vendor

# Set the working directory outside $GOPATH to enable the support for modules.
WORKDIR /src

# Import the code from the context.
COPY ./ ./

# Build the executable to `/app`. Mark the build as statically linked.
RUN go build \
    -installsuffix 'static' \
    -o /app .

# Final stage: the running container.
FROM scratch AS final

# Import the user and group files from the first stage.
COPY --from=builder /user/group /user/passwd /etc/

# Import the Certificate-Authority certificates for enabling HTTPS.
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Import the compiled executable from the second stage.
COPY --from=builder /app /app

# Declare the port on which the webserver will be exposed.
# As we're going to run the executable as an unprivileged user, we can't bind
# to ports below 1024.
EXPOSE 8080

# Perform any further action as an unprivileged user.
USER nobody:nobody

# Run the compiled binary.
ENTRYPOINT ["/app"]
```

go mod vendor

我这个项目有一些外部依赖，在本地开发的时候都已调整好，并且编译通过，在本地开发环境已经生成了两个文件go.mod、go.sum.

在dockerfile的第一步骤中，先启动module模式，且配置代理，因为有些墙外的包服务没有梯子的情况下也是无法下载回来的，这里的代理域名是我自己的，有需要的也可以用。
指令RUN go mod download执行的时候，会构建一层缓存，包含了该项所有的依赖。

之后再次提交的代码中，若是go.mod、go.sum没有变化，就会直接使用该缓存，起到加速构建的作用，也不用重复的去外网下载依赖了。若是这两个文件发生了变化，就会重新构建这个缓存分层。
