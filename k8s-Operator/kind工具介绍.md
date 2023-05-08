[TOC]



# 1.2 Kubernetes集群环境的部署

## 1.2.3 使用Kind 快速搭建Kubernetes环境

https://github.com/kubernetes-sigs/kind

https://kind.sigs.k8s.io

mac

```shell
brew install kind 
kind create cluster --image=kindest/node:v1.22.0 --name dev

kind get clusters 
kind delete clusters kind-2 kind dev

kubectl cluster-info --context kind-kind
kubectl cluster-info --context kind-kind-2
```



## 1.2.4 使用Kind 搭建多节点kubernetes环境

配置文件放在kind-clusters

### **一主多从**

```shell
# vim multi-node-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```



```shell
kind create cluster --config multi-node-config.yaml
```



### **多主多从**

```yaml
# vim ha-config.yaml

# a cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker

```

```shell
kind create cluster --config ha-config.yaml --name dev6
```

## 1.2.5 Kind 用法进阶

### 1. 端口映射

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: "0.0.0.0"
    protocol: tcp
  
```

### 2. 暴露kube-apiserver

kube-apiserver监听127.0.0.1和随机端口，外部机器无法访问，开放外部访问，需做如下修改。

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "192.168.39.1"
```



### 3. 启用Feature Gates

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  FeatureGateName: true
```



### 4. 导入镜像

```
# 假如需要的镜像是my-operator:v1
kind load docker-image my-operator:v1 --name dev
# 假如需要的镜像是一个 tar 包 my-operator.tar
kind load image-archive my-operator.tar --name dev

docker build -t my-operator:v1 ./my-operator-dir
kind load docker-image my-operator:v1
kubectl apply -f apply my-operator.yaml

# 查看kind 环境下有哪些镜像
kubectl get node
docker exec -it dev-control-plane crictl images

crictl -h #查看使用帮助
crictl rmi < image name >  # 删除镜像
```



## 1.3 Kubernetes 集群的基本操作



### 1.3.1 示例项目介绍

https://kubernetes.io/docs/tutorials/stateless-application/guestbook/

### 1.3.2 基础操作演示

1. #### 部署Redis Leader Deployment

```yaml
# vim redis-leader-deployment.yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      containers:
      - name: leader
        image: "docker.io/redis:6.0.5"
        ports:
        - containerPort: 6379

```

```shell
kubectl apply -f redis-leader-deployment.yaml
kubectl get deployment
kubectl get pod
```

2. #### 创建Redis leader Service

```shell
vim redis-leader-service.yaml
```



```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-leader
  labels:
    app: redis
    role: leader
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: leader
    tier: backend
```

```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-service.yaml
kubectl get service
```



3. 部署Redis Follower Deployment

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-deployment.yaml
kubectl get pods
```

4. 创建Redis Follower Service

```shell
vim redis-follower-service.yaml
```



```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: redis-follower
  labels:
    app: redis
    role: follower
    tier: backend
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    app: redis
    role: follower
    tier: backend
```

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-service.yaml
kubectl get service
```

5. **部署Frontend Deployment**

```shell
vim frontend-deployment.yaml
```



```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
        app: guestbook
        tier: frontend
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v5
        env:
        - name: GET_HOSTS_FROM
          value: "dns"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80

```

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
kubectl get deployment
kubectl get pods -l app=guestbook -l tier=frontend
```

6. **创建Frontend Service**

```
vim frontend-service.yaml
```



```yaml
# SOURCE: https://cloud.google.com/kubernetes-engine/docs/tutorials/guestbook
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  #type: LoadBalancer
  ports:
    # the port that this service should serve on
  - port: 80
  selector:
    app: guestbook
    tier: frontend


```

```shell
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml
kubectl apply -f frontend-service.yaml
kubectl get services
```

7. **查看Guestbook效果**

```shell
kubectl port-forward svc/frontend 8080:80
```

