K8s 安装工具



# 安装OpenEBS

```shell
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
kubectl -n openebs get pods
kubectl get sc

# 使用以下命令将 openebs-hostpath 设置为 default，这样在使用 StorageClass 的情况下，如果没有配置 StorageClass，就使用默认的，命令如下：
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 
mkdir -p /home/shiyanlou/Code/devops
kubectl create namespace devops
mkdir /home/shiyanlou/Code/devops/sy-01-1
cd /home/shiyanlou/Code/devops/sy-01-1
```

在 `/home/shiyanlou/Code/devops/sy-01-1` 目录下创建 `nginx-deploy.yaml` 文件，写入以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx:1.8
          imagePullPolicy: IfNotPresent
          name: nginx
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - sleep 15
          resources:
            requests:
              cpu: "0.5"
              memory: 10M
            limits:
              cpu: "0.5"
              memory: 10M
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: http
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 3
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
```

```shell
kubectl apply -f nginx-deploy.yaml
kubectl get po
kubectl set image deployment/nginx nginx=nginx:1.9
# 更新过后，可以使用 kubectl get pod -w 观察升级过程
kubectl get pod -w

# 应用回滚
# 查看历史版本
kubectl rollout history deployment nginx 
# 回滚到上一个版本
kubectl rollout undo deployment nginx
# 指定版本回滚
kubectl rollout undo deployment nginx --to-revision 2
```

单纯的部署好 Deployment 仅仅代表把应用部署到 Kubernetes 中而已，而且通过 Deployment 创建的 Pod 的 IP 地址是可变的，也就是每次更新或者回滚，其 IP 地址可能发生变化，这样就导致其他应用无法稳定的调用某个固定的 IP，基于此，Kubernetes 通过 Service 来为 Pod 提供负载。

Service 的类型有 4 种，Cluster IP，LoadBalance，NodePort，ExternalName。其中 Cluster IP 是默认的类型。

- Cluster IP：通过 集群内部 IP 暴露服务，默认是这个类型，选择该值，这个 Service 服务只能通过集群内部访问；
- LoadBalance：使用云提供商的负载均衡器，可以向外部暴露服务，选择该值，外部的负载均衡器可以路由到 NodePort 服务和 Cluster IP 服务；
- NodePort：顾名思义是 Node 之上的 Port，如果选择该值，这个 Service 可以通过 `NodeIP:NodePort` 访问这个 Service 服务，NodePort 会路由到 Cluster IP 服务，这个 Cluster IP 会通过请求自动创建；
- ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容，没有任何类型代理被创建，可以用于访问集群内其他没有 Labels 的 Pod，也可以访问其他 NameSpace 里的 Service。

在 `/home/shiyanlou/Code/devops/sy-01-1` 目录下创建 `nginx-svc.yaml` 文件，写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
```

```
kubectl apply -f nginx-svc.yaml
kubectl get service
```

可以看到生成的 CLUSTER-IP 为 `10.98.142.116`，在集群内部可以使用该 IP 直接进行访问后端 Pod。但是，该 IP 无法在集群外部访问，如果要访问，就需要把 Service 中的 Type 类型改成 NodePort，如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
```

上面介绍的 Service 主要用在集群内部，当然 NodePort 和 LoadBalancer 类型也可以用于外部访问，但是它们也有一定的弊端：

- 如果使用 NodePort 类型，需要维护好每个应用的端口地址，如果服务太多就不好管理
- 如果使用 LoadBalancer 类型，基本是在云上使用，需要的 IP 比较多，价格也比较昂贵
- 不论是 NodePort 还是 LoadBalancer，都是工作在四层，对于 HTTPS 类型的请求无法直接进行 SSL 校验

因此，社区提供了 Ingress 对象，为集群提供统一的入口，逻辑如下：

![c80ebde4db1520210144d14708fd74b7-0](/Users/cpy/cpy_code/cpy_docs/k8s-二进制部署/k8s应用.assets/c80ebde4db1520210144d14708fd74b7-0.jpg)

其中 Ingress 代理的并不是 Pod 的 Service，而是 Pod，之所以在配置的时候是配置的 Service，是为了获取 Pod 的信息。

如果要使用 Ingress，必须安装 Ingress Controller。我们这里以 `Nginx ingress controller` 为例。

```shell
# 安装 Nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/joker-bai/kubernetes-software-yaml/main/ingress/nginx/ingress-nginx.yaml
```

ingress-controller 会部署在 `ingress-nginx` 名称空间下，所以我们可以使用 `kubectl get pod -n ingress-nginx` 查看部署状态，Pod 的状态为 `running` 则表示部署成功，如下：

#### 暴露 ingress-nginx 服务

在 `/home/shiyanlou/Code/devops/sy-01-1` 目录下创建 `ingress-svc.yaml` 文件，写入以下内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ingress-nginx
  name: ingress-nginx-controller-svc
  namespace: ingress-nginx
spec:
  ports:
    - appProtocol: http
      name: http
      nodePort: 30080
      port: 80
      protocol: TCP
      targetPort: http
    - appProtocol: https
      name: https
      nodePort: 30443
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: NodePort
```

然后执行命令 `` 部署，使用 `` 可以查看 ingress-controller Service 的信息，如下：

```shell
kubectl apply -f ingress-svc.yaml
# 查看 ingress-controller Service 的信息
kubectl get service -n ingress-nginx
# NAME                                 TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
# ingress-nginx-controller             LoadBalancer   10.0.0.131   <pending>     80:32560/TCP,443:30214/TCP   2m9s
# ingress-nginx-controller-admission   ClusterIP      10.0.0.161   <none>        443/TCP                      2m8s
# ingress-nginx-controller-svc         NodePort       10.0.0.109   <none>        80:30080/TCP,443:30443/TCP   6s
```

由于 ingress-controller 也是一个 Pod，它本身也需要暴露出去才能访问，所以如上 Service 可以通过 NodePort 进行访问。

#### 配置 Nginx 应用的 Ingress

在 `/home/shiyanlou/Code/devops/sy-01-1` 目录下创建 `nginx-ingress.yaml` 文件，写入以下内容：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: nginx.devops.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: nginx
                port:
                  number: 80
            pathType: Prefix
```

```shell
# 创建 Ingress
kubectl apply -f nginx-ingress.yaml
# 查看创建状态
kubectl get ingress
```

#### 配置域名解析并访问

但是现在我们无法直接进行访问，因为域名没有做解析，为了方便，直接在本地 hosts 里进行解析。

```
sudo vim /etc/hosts
# 192.168.3.125 nginx.devops.com
# IP 地址是 Kubernetes 集群任意节点 IP，根据实际情况做相应修改。
# 然后打开浏览器，输入 http://nginx.devops.com:30080 进行访问，如下表示能够正常访问：
```

