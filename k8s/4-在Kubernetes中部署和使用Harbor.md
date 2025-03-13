# 在Kubernetes中部署和使用Harbor

## 实验介绍

我们应用镜像最终都会保存到镜像仓库中，有官方的仓库 Dockerhub，也有公有云的仓库，还可以自建镜像仓库。

本次实验就是带大家从 0 到 1 搭建 Harbor 镜像仓库，并完成应用的镜像构建、推送、下载。

#### 知识点

- Harbor 搭建
- Harbor 使用
- Dockerfile 开发
- docker 命令使用

## 前置准备

- 有可用的 Kubernetes 集群
- 安装好 Helm 客户端
- 熟悉 docker 命令的使用
- 熟悉 Dockerfile 的语法规则
- 有可用的 StorageClass

## Harbor 简介

Harbor 是一个用于存储和分发 Docker 镜像的企业级 Registry 服务器，通过添加一些企业必需的功能特性，例如安全、标识和管理等，扩展了开源 Docker Distribution。作为一个企业级私有 Registry 服务器，Harbor 提供了更好的性能和安全。提升用户使用 Registry 构建和运行环境传输镜像的效率。

## 部署 Harbor

这里我们使用 helm 来部署 Harbor。

#### 添加 Harbor 官方的 Helm Chart 仓库

```bash
helm repo add harbor  https://helm.goharbor.io
helm repo update

```

#### 下载 Harbor Charts 包到本地

进入 `~/Code/devops` 目录中，创建一个 `sy-01-4` 目录，并下载 Harbor 的 Chart：

```bash
mkdir -p ~/Code/devops/sy-01-4
cd ~/Code/devops/sy-01-4
helm search repo harbor
helm pull harbor/harbor --untar
# 命令执行后，会把 Harbor 的 Charts 下载到本地并解压，如下：
ll
# drwxr-xr-x 3 root root 111 12月  2 13:34 harbor

```

#### 创建名称空间

创建 `harbor` 名称空间，将 Harbor 相关的应用都部署到该名称空间下。

```bash
kubectl create namespace harbor
```

#### 自定义配置

在 `/home/shiyanlou/Code/devops/sy-01-4/harbor` 目录下创建 `my-values.yaml` 文件，输入以下内容：

```yaml
expose:
  type: nodePort
  tls:
    enabled: false
externalURL: http://192.168.3.125:30002
persistence:
  persistentVolumeClaim:
    registry:
      storageClass: "openebs-hostpath"
    chartmuseum:
      storageClass: "openebs-hostpath"
    jobservice:
      storageClass: "openebs-hostpath"
    database:
      storageClass: "openebs-hostpath"
    redis:
      storageClass: "openebs-hostpath"
    trivy:
      storageClass: "openebs-hostpath"
```

> PS: externalURL: `http://192.168.3.125:30002` 配置的是 Harbor UI 登录的地址，各位在做实验的时候根据自己的配置更改。



```shell
helm upgrade --install harbor harbor/harbor --namespace harbor --create-namespace \
  --set expose.type=ingress \
  --set expose.ingress.className=nginx \
  --set expose.ingress.hosts.core=core.harbor.domain \
  --set expose.ingress.hosts.notary=notary.harbor.domain \
  --set externalURL=https://core.harbor.domain:32466 \
  --set harborAdminPassword="Harbor12345" \
  --set registry.existingSecretKey="S7ZtwtNSC5UA8Lsp"

```



#### 安装 Harbor

使用如下命令进行 Harbor 的安装：

```bash
cd ~/Code/devops/sy-01-4/harbor
helm install harbor -n harbor -f my-values.yaml .

# 查看部署情况，只有当所有应用都是 `running` 的时候才表示部署成功，如下：
kubectl get pod -n harbor
# 查看 Harbor 组件的 Service 信息，如下：
kubectl get service -n harbor
```



## 登录

登录 Harbor 使用我们在 `my-values.yaml` 里配置的 `externalURL` 地址：

输入用户名/密码： `admin/Harbor12345` 进行登录：



## 客户端操作

现在，我们要尝试 Pull/Push 镜像到 Harbor 仓库中。首先我们需要登录 Harbor，命令如下：

```bash
docker login http://10.111.127.141:30002
```

待我们输入用户名/密码： `admin/Harbor12345`，会发现无法登录，报错如下：

http: server gave HTTP response HTTPS client  

这是由于默认 docker registry 使用的是 https，而目前的 Harbor 使用的是 http，我们需要更改 Docker 的配置，使用 `sudo vim /etc/docker/daemon.json`，在里面插入以下内容：

```json
  "insecure-registries": ["http://10.111.127.141:30002"]
```

修改后如下：



然后重载 docker 配置文件，命令如下：

```bash
sudo systemctl daemon-reload
sudo systemctl reload docker
```

> PS：如果使用 `sudo systemctl reload docker` 登录还是失败，就需要使用 `sudo systemctl restart docker` 重启 Docker，重启会导致 Kubernetes 集群重建，耗时会比较久。

现在我们可以继续使用 `docker login http://10.111.127.141:30002` 进行登录，效果如下：

#### 推送镜像到 Harbor 仓库

我们本地现在存在 `nginx:1.8` 的镜像，如下：

该 Nginx 是从 Dockerhub 上拉取的，如果要推送到私有的 Harbor，需要修改镜像名，命令如下：

```bash
docker tag nginx:1.8 10.111.127.141:30002/dev/nginx:1.8
```

然后再使用 `docker push` 命令把镜像推送到 Harbor，命令如下：

```bash
docker push 10.111.127.141:30002/dev/nginx:1.8
```

输出如下：

#### 拉取镜像

拉取镜像的前提也是要登录 Harbor，由于我们已经登录了，这里就可以直接使用命令拉取，如下:

```bash
docker pull 10.111.127.141:30002/dev/nginx:1.8
```

到此，就完成了 Harbor 的基本操作。

# 开发 Dockerfile

Dockerfile 是一个文本文件，里面包含一条条指令，每一条指令就是一层镜像。 一般情况下，Dockerfile 分为 4 个部分：

- 基础镜像
- 维护者信息
- 镜像操作指令
- 容器启动时执行命令

在制作镜像的时候，需要考虑以下几点：

- 命名首字母必须是大写；
- 引用的文件必须在当前的目录及其以下子目录；
- 如果有些文件不需要被打包，可以将这些文件记录到当前目录下隐藏文件（.dockeringore）中；

其重要的命令字段有：

- FROM：指定基础镜像
- COPY：将文件拷贝到容器里
- ADD：将本地文件或者 URL 添加到容器里
- ENV：指定环境变量
- RUN：指定 build 时执行命令
- CMD：指定启动时执行命令
- ENTRYPOINT：指定启动时执行命令
- EXPOSE：指定暴露的端口
- ARG：指定参数，在 `docker build` 时候可以传入
- VOLUME：指定容器挂载点
- LABEL：指定容器标签

> PS: 这里只对 Dockerfile 做了简单的介绍，更多操作需要私下去学习。

现在我们来编写 `go-hello-world` 的 Dockerfile。

由于我们的项目是使用 Go 开发的简单 WEB，这里可以直接使用 `go build` 即可完成应用打包，所以我们的 Dockerfile 可以如下：

```dockerfile
FROM golang
ENV GOPROXY https://goproxy.cn
ADD . /app
WORKDIR /app
RUN go mod tidy
RUN GOOS=linux GOARCH=386 go build -v -o /app/go-hello-word
CMD ["./go-hello-world"]
```

如果拉取依赖包超时，可以把 Dockerfile 改成如下：

```dockerfile
FROM golang
ADD . /app
WORKDIR /app
RUN go env -w GOPROXY=https://goproxy.cn,direct && go mod tidy
RUN GOOS=linux GOARCH=386 go build -v -o /app/go-hello-word
CMD ["./go-hello-world"]
```

Go 开发的应用最后制作出来的是一个二进制可执行文件，如果按照上面的 Dockerfile 进行镜像制作，会把不必要的源代码也打包进去，这就造成了严重的资源浪费，为此，我们可以使用 **多阶段构建**，Dockerfile 改造如下：

```dockerfile
FROM golang AS build-env
ENV GOPROXY https://goproxy.cn
ADD . /go/src/app
WORKDIR /go/src/app
RUN go mod tidy
RUN GOOS=linux GOARCH=386 go build -v -o /go/src/app/go-hello-world

FROM alpine
COPY --from=build-env /go/src/app/go-hello-world /usr/local/bin/go-hello-world
EXPOSE 8080
CMD [ "go-hello-world" ]
```

这样，我们最后制作出来的镜像就只有可执行的二进制文件。

现在，我们可以在源代码根目录下创建一个 `Dockerfile` 文件，写入以上的内容。

然后就可以使用以下命令进行镜像构建：

```bash
docker build -t 192.168.3.175/dev/go-hello-world:v0.0.1 .
```

输出如下：



将镜像推送到 Harbor 仓库，命令如下：

```bash
docker push 192.168.3.175/dev/go-hello-world:v0.0.1
```

# 在 Kubernetes 中部署应用

我们在上一章节中已经为 `go-hello-world` 制作好 Charts 并上传到 Gitlab，现在我们就基于这个 Chart 来部署应用。

现在，我们已经把代码拉取到 `/home/shiyanlou/Code/go-hello-world` 目录下，如果还没有拉取的，可以使用 `git clone` 拉取代码即可。

现在我们进入 `deploy/charts` 目录，使用 `helm install` 部署应用，详细命令如下：

```bash
cd deploy/charts
helm install go-hello-world --set image.repository=192.168.3.175/dev/go-hello-world --set image.tag=v0.0.1 --set containers.port=8080 --set containers.healthCheak.path=/health .
```

然后使用 `kubectl get pod` 查看应用状态，如下：



现在应用已经顺利部署到 Kubernetes 了，但是现在我们还没有对其配置 Ingress 进行访问。先使用 `helm upgrade` 进行更新，命令如下：

```bash
helm upgrade go-hello-world --set image.repository=10.111.127.141:30002/dev/go-hello-world --set image.tag=v0.0.1 --set containers.port=8080 --set containers.healthCheak.path=/health --set ingress.enabled=true --set ingress.hosts[0].host=hello.devops.com --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific .
```

然后使用 `kubectl get ingress` 查看 Ingress 信息，如下：



使用 `sudo vim /etc/hosts` 打开 hosts 文件，在最后添加解析：

```bash
10.111.127.141 hello.devops.com
```

在浏览器访问，显示如下：



如果使用域名访问报 404，则去查看 Ingress 配置是否配置，使用 `kubectl describe ingress go-hello-world` 查看详细配置，查看 annotations 字段有没有 `kubernetes.io/ingress.class: nginx`，如果没有添加上即可。