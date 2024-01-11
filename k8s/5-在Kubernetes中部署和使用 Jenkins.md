5-在Kubernetes中部署和使用 Jenkins

# 实验介绍

本次实验带你在 Kubernetes 中部署 Jenkins，并完成 Jenkins 的初始化操作，然后创建一个简单的流水线测试 Jenkins 是否能正常工作，最后配置如何基于 Kubernetes 实现动态 Slave。

#### 知识点

- Jenkins 安装
- RBAC 认证
- 流水线创建
- 动态 Slave

# Jenkins 简介

Jenkins 是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。 Jenkins 前身是 Hudson，使用 java 语言开发的自动化发布工具。在中大型金融等企业中普遍使用 Jenkins 来作为项目发布工具。 Jenkins 官方提供的插件使 Jenkins 更为强大。

Jenkins 支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序。

Jenkins 的特点如下：

- 开源免费
- 多平台支持（windows/linux/macos）
- 主从分布式架构
- 提供 web 可视化配置管理页面
- 安装配置简单
- 插件资源丰富

# Jenkins 部署

Jenkins 的部署方式很多，主要如下：

- WAR 包独立部署
- RPM 部署
- Docker 部署

为了方便，这里采用 Docker 部署，并且使用 Kubernetes 编排。

在 `/home/shiyanlou/Code/devops/` 目录下创建一个 `sy-02-1` 目录，命令如下：

```bash
mkdir -p /home/shiyanlou/Code/devops/sy-02-1
cd /home/shiyanlou/Code/devops/sy-02-1
```

# 创建 PVC，为 Jenkins 提供数据持久化

在 `/home/shiyanlou/Code/devops/sy-02-1` 目录下创建一个 `jenkins-pvc.yaml` 文件，写入如下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: devops
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-hostpath
  resources:
    requests:
      storage: 5Gi
```

```shell
# 创建 PVC
kubectl apply -f jenkins-pvc.yaml
# persistentvolumeclaim/jenkins-pvc created

# 查看 PVC 状态
kubectl get pvc -n devops jenkins-pvc
# NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS       AGE
# jenkins-pvc   Pending                                      openebs-hostpath   37s

```

# 创建 Jenkins 角色并授权

在 `/home/shiyanlou/Code/devops/sy-02-1` 目录下创建一个 `jenkins-sa.yaml` 文件，写入如下内容：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: devops

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-cr
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-crd
roleRef:
  kind: ClusterRole
  name: jenkins-cr
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: jenkins-sa
    namespace: devops
```



```shell
# 创建 Jenkins 角色并完成授权
kubectl apply -f jenkins-sa.yaml
# serviceaccount/jenkins-sa created
# clusterrole.rbac.authorization.k8s.io/jenkins-cr created
# clusterrolebinding.rbac.authorization.k8s.io/jenkins-crd created
```

# 部署 Jenkins 应用

在 `/home/shiyanlou/Code/devops/sy-02-1` 目录下创建 `jenkins-deploy.yaml` 文件，并写入如下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops
spec:
  selector:
    matchLabels:
      app: jenkins
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccount: jenkins-sa
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          imagePullPolicy: IfNotPresent
          env:
            - name: JAVA_OPTS
              value: -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
          ports:
            - containerPort: 8080
              name: web
              protocol: TCP
            - containerPort: 50000
              name: agent
              protocol: TCP
          resources:
            limits:
              cpu: 1000m
              memory: 1Gi
            requests:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12
          volumeMounts:
            - name: jenkinshome
              mountPath: /var/jenkins_home
      securityContext:
        fsGroup: 1000
      volumes:
        - name: jenkinshome
          persistentVolumeClaim:
            claimName: jenkins-pvc
```



```shell
# 部署 Jenkins
kubectl apply -f jenkins-deploy.yaml
# deployment.apps/jenkins created

# 查看应用部署状态,只有当应用变成 `Running` 的时候才表示部署成功
kubectl get pod -n devops | grep jenkins
# jenkins-689775956-66zcv       0/1     ContainerCreating   0          34s
# jenkins-689775956-66zcv       0/1     Running             0          34s

```

# 创建 Jenkins Service

这里我们使用 NodePort 访问 Jenkins。

在 `/home/shiyanlou/Code/devops/sy-02-1` 目录下创建 `jenkins-svc.yaml` 文件，并写入如下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: devops
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
    - name: web
      port: 8080
      targetPort: web
      nodePort: 30880
    - name: agent
      port: 50000
      targetPort: agent
```

```shell
# 创建 Service
kubectl apply -f jenkins-svc.yaml
# service/jenkins created

# 查看 Service 状态
kubectl get svc -n devops jenkins
# NAME      TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)                          AGE
# jenkins   NodePort   10.0.0.7     <none>        8080:30880/TCP,50000:30299/TCP   28s
```

到此 Jenkins 部署完成。

# Jenkins 初始化

因为我们 Jenkins 是直接使用 Service NodePort 暴露，我们可以直接使用 `http://192.168.3.125:30880` 进行访问，如下：

登录需要获取管理员密钥，可以直接使用 `kubectl logs -n devops [POD_NAME]` 查看密钥，如下：

复制密钥到浏览器上，填写后继续进入插件安装界面，就选择 `安装推荐的插件`，如下：

然后会按照默认的插件，等待安装完成，如下：

然后创建管理员账户，填写具体的信息，点击保存，如下：

配置 Jenkins 地址，默认即可，如下：

然后点击 **开始使用 Jenkins** 即完成 Jenkins 的初始化，进入 Jenkins 界面，如下：

# Jenkins 测试

现在，我们创建一个简单的 `自由风格` 项目，来测试一下 Jenkins 是否工作正常。

首先点击 **新建 item** 创建一个任务。

输入任务名称，选择自由风格

拉到下方 **构建** 处，选择 **执行 Shell**，输入 `echo 'Hello world'`，如下：

点击保存，第一条流水线就创建完成。

点击 **立即构建** 执行流水线。

点击构建过后，会显示构建历史，如下：

选择构建序列，进入后选择 **控制台输出** 可以看到输出日志，如下：

从输出可以知道流水线构建成功，表示 Jenkins 工作正常。

# 配置 Slave

之前在物理机部署 Jenkins 的时候，会使用静态的 Slave，也就是在规定的节点上配置 jenkins-agent，这种方式有一些问题，比如：

- 每个 Slave 的配置环境不一样，来完成不同语言的编译打包等操作，但是这些差异化的配置导致管理起来非常不方便，维护起来也是比较费劲。
- 资源分配不均衡，有的 Slave 要运行的 job 出现排队等待，而有的 Slave 处于空闲状态。
- 资源有浪费，每台 Slave 可能是物理机或者虚拟机，当 Slave 处于空闲状态时，也不会完全释放掉资源。

为此，在 Kubernetes 中，我们采用动态 Slave 的模式，这种模式只有在运行流水线的时候才创建 Slave，流水线运行完成就把 Slave 干掉，逻辑图如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/afd88bc2eef96ab40ad925b315b0cbb8-0.png)

使用这种方式有以下优点：

- 服务高可用，当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的 Jenkins Master 容器，并且将 Volume 分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。
- 动态伸缩，合理使用资源，每次运行 Job 时，会自动创建一个 Jenkins Slave，Job 完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes 会根据每个资源的使用情况，动态分配 Slave 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。
- 扩展性好，当 Kubernetes 集群的资源严重不足而导致 Job 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。

#### 安装 kubernetes 插件

选择 **系统管理** -> **插件管理** -> **可选插件**，搜索 `kubernetes`，选择插件点击安装，如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/1822f94f44592273c971eff1914fc605-0.png)

等待其安装并重启完成。

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/b1dc625b59302f53e9e0a88dced93440-0.png)

#### 配置 Kubernetes 插件

选择 **系统管理** -> **节点管理** -> **Configure Clouds**，配置集群中选择 `kubernetes`，如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/5664c936face3a109eeb69ce9c6fdbea-0.png)

然后配置 Kubernetes 地址（内部地址）和 Slave 运行的名称空间，如下：

https://kubernetes.default.svc.cluster.local

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/f80f1bbe5c4993f09c13f8b17ab59164-0.png)

点击 **连接测试**，查看 Jenkins 和 Kubernetes 集群的通信状态，当结果为 `Connected to Kubernetes xxx` 表示连通性正常，如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/b7b5230bd191261e7e376bebcd582cc0-0.png)

> PS：如果连接测试失败，很可能是权限问题，我们就需要把 ServiceAccount 的凭证 jenkins-sa 添加进来。

添加 Jenkins 地址，填写 Service 地址即可，如下：

http://jenkins.devops.svc.cluster.local:8080

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/aeeabd00c6d1e22100e17f6653db46c0-0.png)

选择 `Pod Templates`，添加 Slave Pod 模板，如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/image-20240110225059002-4898261.png)

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/815e1cb0470831adb8c17a0278ad1293-0.png)

选择 **添加模板**，按需配置，如下：

jenkins-slave

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/42edc650f606d8af602322930a4164b5-0.png)

添加容器模板，镜像 `registry.cn-hangzhou.aliyuncs.com/coolops/jenkins:jnlp6` 里有 Jenkins Slave 客户端，也有 docker 和 kubectl 命令，如下：

> 注意：registry.cn-hangzhou.aliyuncs.com/coolops/jenkins:jnlp6 运行不了。
>
> 使用：jenkins/inbound-agent:3107.v665000b_51092-15-jdk17 正常，但是没有docker和kubectl 指令

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/2b077907c03c01df7cb8b65bde737486-0.png)

其中：

- 运行命令是：`jenkins-slave`
- 命令参数是：`${computer.jnlpmac} ${computer.name}`

另外需要挂载两个主机目录：

1. `/var/run/docker.sock`：该文件是用于 Pod 中的容器能够共享宿主机的 Docker；
2. `/home/shiyanlou/.kube`：这个目录挂载到容器的 `/root/.kube` 目录下面这是为了让我们能够在 Pod 的容器中能够使用 kubectl 工具来访问我们的 Kubernetes 集群，方便我们后面在 Slave Pod 部署 Kubernetes 应用；

如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/eb62cb64ef9a82a18f014680807bcf17-0.png)

最后添加 `Service Account`，为执行操作 Kubernetes 提供权限，如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/72296173e053d07a3195e1c2f307ed81-0.png)

点击 `保存`，即完成 Kubernetes 插件配置。

#### 测试 Kubernetes 插件

回到 Dashboard，选择 **新建任务**，输入 `test-kubernetes-slave`，如下：

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/0faac551da5cf1598e42f0e4c03053c0-0.png)

选择节点标签，也就是在 Kubernetes 插件中配置的 Label，如下：

jenkins-jnlp

![图片描述](/Users/cpy/cpy_code/cpy_docs/k8s/5-在Kubernetes中部署和使用 Jenkins.assets/9838bbb94a31322ce4e9eb4c31b59cc3-0.png)

在 **构建** 步骤中，选择 **执行 Shell**，在文本中输入以下内容：

```bash
echo "test docker command"
docker info

echo "------------"

echo "test kubectl command"
kubectl get pod
```

如下：

然后点击保存即完成项目配置。

先打开 **终端**，输入 `kubectl get pod -n devops -w` 观察 Slave 创建情况。

再次点击 **立即构建**，如下：



然后在终端可以看到 jenkins-slave 从创建到删除的整个过程，如下：

在 jenkins 面板也能看到构建历史，如下：

点击构建历史中的任务，选择 **控制台输出**，则可看到具体的构建日志，如下：

现在，我们已经完成基于 Kubernetes 生成动态 Slave 的配置了。

# 实验总结

现在我们对整个实验进行总结。

首先带大家了解什么是 Jenkins，以及如何在 Kubernetes 中部署 Jenkins，包括数据持久化，应用访问等，再然后带大家完成 Jenkins 的初始化并测试流水线功能是否正常，最后，为了最大化的利用 Kubernetes 的特性，带大家配置如何基于 Kubernetes 实现 Jenkins Slave 的动态化，通过模板定制、标签关联可以实现不同任务关联不同的 Slave，再企业中也可以根据实际情况配置多个模板，比如 Java 应用的模板，Nodejs 的模板等。