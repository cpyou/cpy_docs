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