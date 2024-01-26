# 在Kubernetes中部署和使用Gitlab

[TOC]



# 实验介绍

Gitlab 是代码仓库，其部署方式有很多，企业级使用是找单独的机器部署管理，这里为了实验方便，将演示如何在 Kubernetes 中部署 Gitlab，其组件主要涉及 Redis、PostgreSQL 和 Gitlab，并且会演示如何将代码仓库进行持久化。

**知识点**

- PV 创建
- Redis 部署
- PostgreSQL 部署
- Gitlab 部署
- Deployment 开发
- Service 使用

# 前置准备

- 有可用的 Kubernetes 集群
- Kubernetes 有持久化存储
- 熟悉 Kubernetes 的基础操作

在 `/home/shiyanlou/Code/devops` 目录下创建一个 `sy-01-2` 目录，本次实验的所有配置文件会保存到该文件夹下，命令如下：

```bash
mkdir -p /home/shiyanlou/Code/devops/sy-01-2
cd /home/shiyanlou/Code/devops/sy-01-2
```

在第一个实验中，我们已经初始化好 Kubernetes 环境，并且环境自带一个 `local` 存储，也创建好了 `StroageClass`，如下：

```bash
# 查看local存储
kubectl get sc
# NAME                         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
# openebs-device               openebs.io/local   Delete          WaitForFirstConsumer   false                  18d
# openebs-hostpath (default)   openebs.io/local   Delete          WaitForFirstConsumer   false                  18d
```

# 部署redis

## 创建 Redis 持久化存储

Redis 主要用作缓存，在 Gitlab 的整个体系中也是一样，虽然是基于内存的数据库，但是为了保持服务的可靠性，其数据也会保存到本地，不管是 RDB 模式还是 AOF 模式，所以对于 Redis，我们在部署的时候也要做好数据持久化，以便后期维护。

在 `~/Code/devops/sy-01-2` 目录下创建一个 `redis-pvc.yaml` 文件，并写入以下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: devops
spec:
  accessModes:
    - ReadWriteOnce
  # 这里指定使用的 OpenEBS 的 sc
  storageClassName: openebs-hostpath
  resources:
    requests:
      storage: 5Gi
```



```shell
# 创建 PVC
kubectl create namespace devops
kubectl apply -f redis-pvc.yaml
# 查看创建情况 如下表示创建成功
kubectl get pvc -n devops redis-pvc
# NAME        STATUS   VOLUME  CAPACITY   ACCESS MODES   STORAGECLASS       AGE
# redis-pvc   Pending                                    openebs-hostpath   17s

# 创建成功显示
# NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
# redis-pvc   Bound    pvc-02e0bdbd-cd5e-46ab-b188-5f3795453545   5Gi        RWO            openebs-hostpath   1d
```

PS: PVC 的状态之所以是 Pending，是因为该集群的 StroageClass 的绑定模式是 WaitForFirstConsumer，在该模式下，只有相关的 Pod 使用对应的 PVC 的时候才会真正创建。可以通过 `kubectl get sc openebs-hostpath -oyaml` 来查看 StorageClass 的详细配置。

## 创建 Redis 服务

在 `~/Code/devops/sy-01-2` 目录下创建一个 `redis-deploy.yaml` 文件，并写入以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: devops
  labels:
    name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      name: redis
  template:
    metadata:
      name: redis
      labels:
        name: redis
    spec:
      containers:
        - name: redis
          image: sameersbn/redis
          imagePullPolicy: IfNotPresent
          ports:
            - name: redis
              containerPort: 6379
          volumeMounts:
            - mountPath: /var/lib/redis
              name: data
          livenessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 30
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 5
            timeoutSeconds: 1
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: redis-pvc
```

```shell
# 部署 Redis
kubectl apply -f redis-deploy.yaml

#  查看 Redis 部署情况，当状态变为 running 的时候表示部署成功，如下：
kubectl get po -n devops | grep redis
# redis-65ccf97bcb-kslv5   1/1     Running   1          48s
```

## 创建 Redis Service

在 `~/Code/devops/sy-01-2` 目录下创建一个 `redis-svc.yaml` 文件，并写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: devops
  labels:
    name: redis
spec:
  ports:
    - name: redis
      port: 6379
      targetPort: redis
  selector:
    name: redis
```

```shell
# 创建 Service
kubectl apply -f redis-svc.yaml

# 查看创建情况
kubectl get svc -n devops redis
# NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
# redis   ClusterIP   10.0.0.211   <none>        6379/TCP   6s
```

> PS: 由于在本次实验中，Redis 的 Service 的类型是 ClusterIP，所以只能在集群内部访问，如果特殊原因需要集群外部访问，可以自行将类型改成 NodePort。

```shell
# 查看 Service 的具体信息
kubectl describe svc -n devops redis
# Name:              redis
# Namespace:         devops
# Labels:            name=redis
# Annotations:       <none>
# Selector:          name=redis
# Type:              ClusterIP
# IP Families:       <none>
# IP:                10.0.0.211
# IPs:               10.0.0.211
# Port:              redis  6379/TCP
# TargetPort:        redis/TCP
# Endpoints:         172.16.169.163:6379
# Session Affinity:  None
# Events:            <none>
# 至此，Redis 部署完成。
```

# 部署 PostgreSQL

PostgreSQL 是 Gitlab 的默认数据库，保存除 Repo、wiki 以及 ssh key 之外的所有数据，包括账户信息、配置、issue 等。所以它在 Gitlab 的体系中占据很重要的地位。

## 创建 PostgreSQL 持久化存储

在 `~/Code/devops/sy-01-2` 目录下创建一个 `postgresql-pvc.yaml` 文件，并写入以下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pvc
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
kubectl apply -f postgresql-pvc.yaml

# 查看 PVC 创建情况，如下表示创建成功
kubectl get pvc -n devops postgresql-pvc
# NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS       AGE
# postgresql-pvc   Pending                                      openebs-hostpath   7s
```

## 创建 PostgreSQL 应用

在 `~/Code/devops/sy-01-2` 目录下创建一个 `postgresql-deploy.yaml` 文件，并写入以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: devops
  labels:
    name: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      name: postgresql
  template:
    metadata:
      name: postgresql
      labels:
        name: postgresql
    spec:
      containers:
        - name: postgresql
          image: sameersbn/postgresql:10
          imagePullPolicy: IfNotPresent
          env:
            - name: DB_USER
              value: gitlab
            - name: DB_PASS
              value: passw0rd
            - name: DB_NAME
              value: gitlab_production
            - name: DB_EXTENSION
              value: pg_trgm
          ports:
            - name: postgres
              containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql
              name: data
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -h
                - localhost
                - -U
                - postgres
            initialDelaySeconds: 30
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -h
                - localhost
                - -U
                - postgres
            initialDelaySeconds: 5
            timeoutSeconds: 1
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgresql-pvc
```

```shell
# 创建 PostgreSQL
kubectl apply -f postgresql-deploy.yaml

# 查看创建情况，只有当 Pod 的状态变成 running 的时候才表示创建成功
kubectl get po -n devops | grep postgresql
# postgresql-5f9d9c7879-pd4tx   0/1     Pending   0          5s
kubectl get po -n devops | grep postgresql
# postgresql-5f9d9c7879-pd4tx   1/1     Running   0          2m28s
```

## 创建 PostgreSQL Service

在 `~/Code/devops/sy-01-2` 目录下创建一个 `postgresql-svc.yaml` 文件，并写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: devops
  labels:
    name: postgresql
spec:
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
  selector:
    name: postgresql
```

然后使用 `kubectl apply -f postgresql-svc.yaml` 创建 Service，再使用 `kubectl get svc -n devops postgresql` 查看创建情况，如下：

```shell
# 创建 Service
kubectl apply -f postgresql-svc.yaml

# 查看创建情况
kubectl get svc -n devops postgresql
# NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
# postgresql   ClusterIP   10.0.0.247   <none>        5432/TCP   6s

# 查看 Service 的详情
kubectl describe svc -n devops postgresql
# Name:              postgresql
# Namespace:         devops
# Labels:            name=postgresql
# Annotations:       <none>
# Selector:          name=postgresql
# Type:              ClusterIP
# IP Families:       <none>
# IP:                10.0.0.247
# IPs:               10.0.0.247
# Port:              postgres  5432/TCP
# TargetPort:        postgres/TCP
# Endpoints:         172.16.36.110:5432
# Session Affinity:  None
# Events:            <none>
```

# 部署Gitlab

## 创建 Gitlab 持久化存储

在 `~/Code/devops/sy-01-2` 目录下创建一个 `gitlab-pvc.yaml` 文件，并写入以下内容：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-pvc
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
kubectl apply -f gitlab-pvc.yaml
# 查看创建结果，如下表示创建成功：
kubectl get pvc -n devops gitlab-pvc
# NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS       AGE
# gitlab-pvc   Pending                                      openebs-hostpath   5s
```

## 部署 Gitlab 应用

在 `~/Code/devops/sy-01-2` 目录下创建一个 `gitlab-deploy.yaml` 文件，并写入以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
  namespace: devops
  labels:
    name: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab
  template:
    metadata:
      name: gitlab
      labels:
        name: gitlab
    spec:
      containers:
        - name: gitlab
          image: sameersbn/gitlab:11.8.1
          imagePullPolicy: IfNotPresent
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: GITLAB_TIMEZONE
              value: Beijing
            - name: GITLAB_SECRETS_DB_KEY_BASE
              value: long-and-random-alpha-numeric-string
            - name: GITLAB_SECRETS_SECRET_KEY_BASE
              value: long-and-random-alpha-numeric-string
            - name: GITLAB_SECRETS_OTP_KEY_BASE
              value: long-and-random-alpha-numeric-string
            - name: GITLAB_ROOT_PASSWORD
              value: admin321
            - name: GITLAB_ROOT_EMAIL
              value: devops@163.com
            - name: GITLAB_HOST
              value: 192.168.3.125
            - name: GITLAB_PORT
              value: "30180"
            - name: GITLAB_SSH_PORT
              value: "30022"
            - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
              value: "true"
            - name: GITLAB_NOTIFY_PUSHER
              value: "false"
            - name: GITLAB_BACKUP_SCHEDULE
              value: daily
            - name: GITLAB_BACKUP_TIME
              value: 01:00
            - name: DB_TYPE
              value: postgres
            - name: DB_HOST
              value: postgresql
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              value: gitlab
            - name: DB_PASS
              value: passw0rd
            - name: DB_NAME
              value: gitlab_production
            - name: REDIS_HOST
              value: redis
            - name: REDIS_PORT
              value: "6379"
          ports:
            - name: http
              containerPort: 80
            - name: ssh
              containerPort: 22
          volumeMounts:
            - mountPath: /home/git/data
              name: data
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 180
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            timeoutSeconds: 1
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: gitlab-pvc
```

其中：

- GITLAB_ROOT_PASSWORD 用于配置 root 用户登录的密码
- GITLAB_HOST 是 gitlab 的地址
- GITLAB_PORT 是 gitlab http 端口
- GITLAB_SSH_PORT 是 gitlab ssh 端口

> PS: 由于我们会直接使用 NodePort 进行访问，所以这里填的地址是 Kubernetes 集群节点地址，端口是 NodePort 暴露的端口，大家在实验的时候根据自己节点 IP 做调整。

```shell
# 创建 Gitlab
kubectl apply -f gitlab-deploy.yaml

# 查看 Pod 创建情况，只有当 Pod 的状态变为 running 的时候，才表示创建成功，如下：
kubectl get po -n devops | grep gitlab
# gitlab-84b4c95478-9pgns       0/1     ContainerCreating   0          7s
# gitlab-84b4c95478-9pgns       1/1     Running   0          11m
```

## 创建 Gitlab Service

在 `~/Code/devops/sy-01-2` 目录下创建一个 `gitlab-svc.yaml` 文件，并写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitlab
  namespace: devops
  labels:
    name: gitlab
spec:
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30180
    - name: ssh
      port: 22
      targetPort: ssh
      nodePort: 30022
  selector:
    name: gitlab
  type: NodePort
```

> PS: Gitlab 的 Service 在这里使用的是 NodePort，因为我们需要从外部进行访问。

```shell
# 创建 Service
kubectl apply -f gitlab-svc.yaml

# 查看 Service 创建情况，如下：
kubectl get svc -n devops gitlab
# NAME     TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)                     AGE
# gitlab   NodePort   10.0.0.215   <none>        80:30180/TCP,22:30022/TCP   5s

# 查看 Service 的详情
kubectl describe svc -n devops gitlab
```

# 使用 Gitlab

## 登录

上面已经把 Gitlab 部署完成了，我们这里直接使用 `NodeIP:NodePort` 进行访问，之所以没有用 Ingress，主要是为了方便后面的实验。NodeIP 可以通过 `kubectl get node -o wide` 获取，NodePort 是自己配置的端口 `30180`，在浏览器输入 `http://192.168.3.125:30180` 进行访问即可。

http://192.168.3.125:30180

然后输入用户名/密码： `root/admin321` 登录 Gitlab

## 创建用户

在生产环境里使用，为了防止单账户权限过高造成的安全问题，很少直接使用超管账户，基本都会创建一些子账户。

比如在这里我们创建一个 dev 账户来使用。点击 Admin Area，选择 `New user`，

## 修改密码

如果配置好邮箱发送信息，会给您发邮件配置密码，不过，由于这里没有配置邮箱信息，我们手动修改密码。

首先通过 `Admin Area - Users` 进入用户管理界面，找到对应的用户，选择 `Edit` 进行修改，如下：

然后输入要更改的密码，比如我这里输入的是 `dev@123456`，如下：

> PS: 默认情况下，Gitlab 要求密码强度大于 8 位。

最后拉到最下方，点击 `Save changes`，即完成密码修改。

## 创建项目

(1) 在 Gitlab 上创建新项目，在首页选择 `Create a project`，如下：

然后根据实际情况填写信息，最后点击 `Create project` 创建项目即可，如下：

这种创建方式默认会把项目直接创建在当前用户下，而在生产级别，公司应用是不会放到个人用户下，所以我们会先创建 Group，如下：

填写组名 `devops`，点击创建即可，如下：

然后再在组里创建项目，具体步骤和上面一样。

## 用户授权

用户创建过后，只有当前账户下的项目管理权限，如果要管理其他项目或者组，就要进行授权。

#### 授权给组权限

首先，进入我们创建的组 `devops`，然后选择左边的 member，如下：

然后添加用户、配置权限、添加到组，如下：

这样 `dev` 用户就拥有 `devops` 组的 `Developer` 权限。

#### 授权给某个项目

有时候只想给用户授权某个项目的权限，直接进入项目，比如我这里进入 hello-word 项目，然后在设置里选择 member，如下：

然后选择 用户、设置权限、添加到项目即可，如下：

## 导入项目

上面介绍的都是自己创建项目，而在整个实验中，为了方便，会先把代码准备好，我们可以直接导入即可。

为此，我准备了一个 Go 项目，后续的流水线都是基于该项目进行，项目地址是：`https://gitee.com/coolops/go-hello-world.git`。

下面我们将该项目导入到 Gitlab 中。

首先在 Gitlab 上创建项目，然后选择 Import project。如下：

然后选择代码源，我这里是 gitee 上的代码，所以选择 `repo by url`,填写对应的信息，如下：

点击导入按钮，等待一段时间，即可完成项目导入。

Gitlab 的使用操作就介绍到这里，更多的功能需要自己去挖掘，目前介绍的功能足够我们完成本课程的后续实验了。

# 实验总结

Gitlab 项目的整个部署包括 Redis、PostgreSQL、Gitlab 三个主要的组件，每个组件的部署过程差不多，都是创建存储卷 PVC，然后使用 Deployment 进行部署服务，最后使用 Service 进行应用访问，当然在正式环境中，如果要把有状态的服务部署到 Kubernetes 中，建议使用 StatefulSet 或者 Operator 的方式部署，这样才能更好的管理有状态应用。

Gitlab 是代码仓库，会保存我们整个实验的源代码，包括应用代码，YAML 清单等。所以请记住其地址以及用户名密码，后面是时常使用到。