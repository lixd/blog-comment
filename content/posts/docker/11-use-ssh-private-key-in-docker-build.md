---
title: "Docker教程(十一)---如何在 Docker Build 时使用 SSH 私钥进行认证"
description: "如何在 Docker Build 时使用 SSH 私钥进行认证"
date: 2023-03-04 22:00:00
draft: false
tags: ["Docker"]
categories: ["Docker"]
---

本文主要通过如何在 Docker Build 时使用 SSH 私钥进行认证，比如拉取私有仓库时就很有用。包括18.09版本之前的使用`多阶段构建`方式，以及 18.09版本后的 `--ssh` 方式。

<!--more-->



## 1. 概述

在实际工作中，Build Docker 镜像时，经常碰上需要在 Docker 镜像内用到 SSH Private Key 的场景。比如构建镜像时要从 GitHub、GitLab 的私有库 Clone 代码，或者要安装私有库的 Gem、NPM Package 等。

而**如果直接把自己的 SSH Private Key 打包到 Docker 镜像中的话，是存在很大安全风险的**。如何解决这个问题？



## 2. 多阶段构建方式

通过参数将私钥传递到容器里，同时配合多阶段构建以解决直接把私钥打包进容器带来的安全风险。

> 使用多阶段构建，只要私钥不出现在最后一阶段，都是比较安全的，中间过程的镜像只会存放在本机，不会公开，因此问题也不大。

Dockerfile 如下：

分为三个阶段：

* 阶段一：拿到私钥并写入到/root/.ssh/id_rsa文件用于认证
* 阶段二：yarn build
* 阶段三：将编译好的产物 COPY 到 node 环境运行

```bash
# Stage 1: get sources from npm and git over ssh
FROM node:carbon AS sources
ARG SSH_KEY
ARG SSH_KEY_PASSPHRASE
RUN mkdir -p /root/.ssh && \
    chmod 0700 /root/.ssh && \
    ssh-keyscan bitbucket.org > /root/.ssh/known_hosts && \
    echo "${SSH_KEY}" > /root/.ssh/id_rsa && \
    chmod 600 /root/.ssh/id_rsa
WORKDIR /app/
COPY package*.json yarn.lock /app/
RUN eval `ssh-agent -s` && \
    printf "${SSH_KEY_PASSPHRASE}\n" | ssh-add $HOME/.ssh/id_rsa && \
    yarn --pure-lockfile --mutex file --network-concurrency 1 && \
    rm -rf /root/.ssh/

# Stage 2: build minified production code
FROM node:carbon AS production
WORKDIR /app/
COPY --from=sources /app/ /app/
COPY . /app/
RUN yarn build:prod

# Stage 3: include only built production files and host them with Node Express server
FROM node:carbon
WORKDIR /app/
RUN yarn add express
COPY --from=production /app/dist/ /app/dist/
COPY server.js /app/
EXPOSE 33330
CMD ["node", "server.js"]
```

build 命令

```bash
docker build -t ezze/geoport:0.6.0 \
  --build-arg SSH_KEY="$(cat ~/.ssh/id_rsa)" \
  --build-arg SSH_KEY_PASSPHRASE="my_super_secret" \
  ./
```



## 3.  SSH mount type

同时 Docker 在 `18.09` 版本后，推出了 BuildKit 的 **SSH mount type**，我们也可以用这个特性来解决该问题。

### Enable BuildKit

由于是 BuildKit 的特性，因此需要设置这个环境来开启 BuildKit

```bash
export DOCKER_BUILDKIT=1
```

或者修改 /etc/docker/daemon.json 文件并重启 docker 服务永久开启 BuildKit，添加内容如下所示：

```bash
{
  "features": {
    "buildkit" : true
  }
}
```



### Dockerfile 修改

首先需要在 Dockerfile 首行开启特性：

```Docke
# syntax=docker/dockerfile:1
```

这句话意思是用 `docker/dockerfile:1` 这个镜像来解析 Dockerfile

然后添加下面的内容，下载对应网站的公钥：

```dockerfile
# Download public key for github.com
RUN --mount=type=ssh mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
```

> 注意替换域名

然后，在 Dockerfile 中需要使用 SSH Private Key 的地方都加上`--mount=type=ssh`

> 这个 flag 指定该命令运行时有权限访问对应的 ssh 私钥，即：其他没有指定的命令是无法使用该私钥的。

比如 go 下载依赖就像这样：

```dockerfile
RUN --mount=type=ssh go mod download
```

比如，Rails 项目安装有私有库的 Gem 包时，就写成这样：

```dockerfile
RUN --mount=type=ssh bundle install
```



### Docker build 命令

然后 docker build 时通过` --ssh`指定私钥：

```bash
docker build -f Dockerfile -t helloworld:1.0.0 . --ssh default=/root/.ssh/id_rsa
```

这种方式，既能正常使用上 SSH Private Key，又能使其在镜像中不留痕迹。完美！



### 完整版

完整 Dockerfile 内容如下：

```Dockerfile
# syntax=docker/dockerfile:1

# Build the manager binary
FROM golang:1.19 as builder
ARG TARGETOS
ARG TARGETARCH

ENV GOPROXY=https://goproxy.cn

WORKDIR /workspace
# Copy the go source
COPY . /workspace
# Download public key for github.com
RUN --mount=type=ssh mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
RUN --mount=type=ssh  go mod download

# Build
# the GOARCH has not a default value to allow the binary be built according to the host where the command
# was called. For example, if we call make docker-build in a local env which has the Apple Silicon M1 SO
# the docker BUILDPLATFORM arg will be linux/arm64 when for Apple x86 it will be linux/amd64. Therefore,
# by leaving it empty we can ensure that the container and binary shipped on it will have the same platform.
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} go build -a -o manager main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM gcr.io/distroless/static:nonroot

WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532

ENTRYPOINT ["/manager"]
```

使用以下命令进行构建：

```bash
docker build -t ${IMG} . --ssh default=~/.ssh/id_rsa
```



### FAQ

**关闭 SSH 严格模式**

在测试时出现以下错误：

```text
#0 0.831   Host key verification failed.
#0 0.831   fatal: Could not read from remote repository.
```

这是因为 ssh 不能识别远程主机提供的秘钥，默认情况下会询问是否信任该秘钥，但是在非交互的环境里会直接拒绝掉。

Dockerfile 里的 `ssh-keycan github.com > known_hosts` 这句就是添加到信任列表，也有可能没生效。可以试试加上下面这句，直接关闭 SSH 的严格模式：

```dockerfile
# Configure ssh to trust unknown host keys:
RUN sed /^StrictHostKeyChecking/d /etc/ssh/ssh_config; \
  echo StrictHostKeyChecking no >> /etc/ssh/ssh_config
```



**buildx**

使用 docker buildx 进行多架构编译时也可以使用同样的方式对 Dockerfile 进行修改，然后在 build 时指定私钥。

```bash
docker buildx build --push --platform "linux/amd64,linux/arm64" -t helloworld:1.0.0  --ssh default=~/.ssh/id_rsa  -f cross.Dockerfile .  
```



**多私钥**

如果有多个私有仓库并且需要不同的私钥进行认证的话也是支持的，需要额外处理一下。

首先是 Dockerfile 里每条命令需要指定使用的私钥

```dockerfile
RUN --mount=type=ssh,id=github_ssh_key go mod download
RUN --mount=type=ssh,id=gitlab_ssh_key bundle install
```

相应的需要信任多个 hosts

```dockerfile
ssh-keyscan github.com >> ~/.ssh/known_hosts
ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
```

最后 build 时需要传递多个私钥

```bash
docker build --ssh github_ssh_key=/path/to/.ssh/github_ssh_id_rsa --ssh gitlab_ssh_key=/path/to/.ssh/gitlab_ssh_id_rsa .
```



## 4. 小结

在 Docker build 时使用 SSH 私钥进行认证有两种比较好的解决方案：

* 1）Docker 18.09 之前的版本，使用多阶段构建，将私钥以参数形式传递进容器，需要保证私钥不出现在最后一阶段即可
* 2）Docker 18.09 及以后，原生支持 `--ssh` 参数，推荐使用

如果不使用 Docker 的话就只能用方案一了。



## 5. 参考

[using-ssh-keys-inside-docker-container](https://stackoverflow.com/questions/18136389/using-ssh-keys-inside-docker-container)

[docker 官方文档](https://docs.docker.com/engine/reference/commandline/buildx_build/#ssh)

[dockerfile-run-mount-type-ssh-doesnt-seem-to-work](https://stackoverflow.com/questions/73263731/dockerfile-run-mount-type-ssh-doesnt-seem-to-work)

[Build secrets and SSH forwarding in Docker 18.09](https://medium.com/@tonistiigi/build-secrets-and-ssh-forwarding-in-docker-18-09-ae8161d066)
