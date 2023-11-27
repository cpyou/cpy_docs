Helm 使用



当项目比较少的时候，这样管理很方便，如果项目比较多或者有些项目比较复杂，会有很多的资源描述文件，例如微服务架构应用，组成应用的服务可能多达十个，几十个。如果有更新或回滚应用的需求，可能要修改和维护所涉及的大量资源文件，而这种组织和管理应用的方式就显得力不从心了。



Helm 有几个重要概念：

- helm：一个命令行客户端工具，主要用于 Kubernetes 应用 chart 的创建、打包、发布和管理。
- Chart：应用描述，一系列用于描述 k8s 资源相关文件的集合。
- Release：基于 Chart 的部署实体，一个 chart 被 Helm 运行后将会生成对应的一个 release；将在 k8s 中创建出真实运行的资源对象。

Helm 客户端下载地址：`https://github.com/helm/helm/releases`

选择自己适合的版本进行下载，比如我这里下载最新的 `v3.9.2` 版本，命令如下：

```
wget https://get.helm.sh/helm-v3.9.2-linux-amd64.tar.gz
tar xf helm-v3.9.2-linux-amd64.tar.gz
cd linux-amd64
sudo mv helm /usr/local/bin
# 查看能否正常获取 Helm 版本
helm version
```

# Helm命令

Helm 的命令很多，最常用的命令如下：

- create：创建 helm chart
- history：查看 release 历史版本
- install：安装一个 Helm Chart
- list：列出集群中的 Release
- package：将 Chart 目录打包
- pull：从远程仓库下载 Chart
- repo：添加，列出，移除，更新和索引 chart 仓库
- rollback：版本回滚
- search：在仓库中搜索 Chart
- status：查看 Release 状态
- uninstall：卸载 Release
- upgrade：更新 Release
- version：查看版本

# 仓库管理



Helm Chart 包可以保存到本地的 Gitlab 仓库，这种方式可以直接以目录的方式存放，就像代码一样，也可以直接放压缩包。

除此之外，还可以保存到一些专门的 Chart 仓库，目前主流的用：

- 微软仓库：`http://mirror.azure.cn/kubernetes/charts/`
- 阿里云仓库：`https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts`
- 官方仓库：`https://hub.kubeapps.com/charts/incubator`

# 添加仓库

添加仓库用 `helm repo add` 命令，如下：

```shell
helm repo add stable http://mirror.azure.cn/kubernetes/charts
# "stable" has been added to your repositories

helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
# "aliyun" has been added to your repositories

# 更新仓库
helm repo update 
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "aliyun" chart repository
# ...Successfully got an update from the "stable" chart repository
# Update Complete. ⎈Happy Helming!⎈

# 删除仓库
helm repo remove aliyun
```

# 制作 Helm Chart

在 `/home/shiyanlou/Code/devops` 目录下创建一个 `sy-01-3` 文件夹，命令如下：

```bash
mkdir -p /home/shiyanlou/Code/devops/sy-01-3
cd /home/shiyanlou/Code/devops/sy-01-3
```

然后使用 `helm create` 创建一个 `go-hello-word` chart，命令如下：

```bash
helm create go-hello-world

tree go-hello-world/
# go-hello-world/
# ├── charts
# ├── Chart.yaml
# ├── templates
# │   ├── deployment.yaml
# │   ├── _helpers.tpl
# │   ├── hpa.yaml
# │   ├── ingress.yaml
# │   ├── NOTES.txt
# │   ├── serviceaccount.yaml
# │   ├── service.yaml
# │   └── tests
# │       └── test-connection.yaml
# └── values.yaml
# 
# 3 directories, 10 files
```

其中：

- Chart.yaml：用于描述这个 Chart 的基本信息，包括名字、描述信息以及版本等。
- values.yaml ：用于存储 templates 目录中模板文件中用到变量的值。
- Templates： 目录里面存放所有 yaml 模板文件。
- charts：目录里存放这个 chart 依赖的所有子 chart。
- NOTES.txt ：用于介绍 Chart 帮助信息， helm install 部署后展示给用户。例如：如何使用这个 Chart、列出缺省的设置等。
- _helpers.tpl：放置模板助手的地方，可以在整个 chart 中重复使用。

其实，这就完成 Chart 的制作，默认会创建一个 Nginx 的 Chart，是不是很简单？

但是，在实际工作中，肯定会在此基础上做调整，我们这里也需要做一些调整，在 `go-hello-world` 中，应用端口是 `8080`，健康检测接口是 `/health`。为了方便，这里将这些字段配置为变量，方便适配其他类型的应用。

- 修改 `go-hello-world/templates/deployment.yaml`，修改部分如下：

  ```
   36           ports:
   37             - name: http
   38               containerPort: 80
   39               protocol: TCP
   40           livenessProbe:
   41             httpGet:
   42               path: /
   43               port: http
   44           readinessProbe:
   45             httpGet:
   46               path: /
   47               port: http
   48           resources:
   -------------------------------------
   36           ports:
   37             - name: http
   38               containerPort: {{ .Values.containers.port }}
   39               protocol: TCP
   40           {{- if .Values.containers.healthCheck.enabled}}
   41           livenessProbe:
   42             httpGet:
   43               path: {{ .Values.containers.healthCheck.path }}
   44               port: http
   45           readinessProbe:
   46             httpGet:
   47               path: {{ .Values.containers.healthCheck.path }}
   48               port: http
   49           {{- end}}
  ```

  

- 修改 `go-hello-world/values.yaml`，修改后如下：

  ```
   13 imagePullSecrets: []
   14 nameOverride: ""
   15 fullnameOverride: ""
   16
   17 containers:
   18   port: 80
   19   healthCheck:
   20     enabled: true
   21     path: /
   22
   23 serviceAccount:
   24   # Specifies whether a service account should be created
  ```

到此，Helm Chart 修改完成，可以将其保存到代码仓库 `go-hello-world` 项目下。

使用 Helm 部署应用非常简单，直接使用 `helm install` 即可，而且其可以跟不同的参数，如果不知道怎么使用，可以用 `helm install -h` 查看帮助文档。

下面我们部署 Nginx 服务，版本是 1.8，命令如下：

```bash
cd /home/shiyanlou/Code/devops/sy-01-3/
helm install nginx --set image.repository=nginx --set image.tag=1.8 go-hello-world/
# NAME: nginx
# LAST DEPLOYED: Wed Nov 15 09:25:52 2023
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# NOTES:
# 1. Get the application URL by running these commands:
#   export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=go-hello-world,app.kubernetes.io/instance=nginx" -o jsonpath="{.items[0].metadata.name}")
#   export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
#   echo "Visit http://127.0.0.1:8080 to use your application"
#   kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT

kubectl get pod
# nginx-go-hello-world-5984f9f589-cjp24              1/1     Running       0          39s

# 可以查看集群中已经部署的 Release
helm list 
# nginx	default  	1       	2023-11-15 09:25:52.377329307 -0500 EST	deployed	go-hello-world-0.1.0	1.16.0
```

如果应用进行了迭代，现在要更新到新的版本，比如我们要更新到 Nginx 1.9 版本，则使用以下命令：

```bash
helm upgrade nginx --set image.repository=nginx --set image.tag=1.9 go-hello-world/

kubectl get po
# nginx-go-hello-world-5984f9f589-cjp24              1/1     Running       0          6m2s
# nginx-go-hello-world-6464d479b8-j9mtv              0/1     Running       0          74s

helm list
# NAME 	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
# nginx	default  	2       	2023-11-15 09:30:40.616751666 -0500 EST	deployed	go-hello-world-0.1.0	1.16.0
# 看到版本号 REVISION 变成了 2

# 直接回滚到上一个版本
helm rollback nginx
# 回滚到指定的版本
helm rollback nginx 1
# 查看历史版本 helm history RELEASE_NAME 
helm history nginx 
# 卸载应用
helm uninstall nginx
```

当然，Helm 可以很简单，也可以很复杂，如果对它不是很了解，可以到官网 `https://helm.sh` 进行进一步的学习