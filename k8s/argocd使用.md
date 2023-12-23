argocd 使用



```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# 删除argocd
kubectl delete ns argocd

# 网络不好
kubectl apply -n argocd -f https://gitee.com/coolops/kubernetes-devops-course/raw/master/argocd/install.yaml

# 查看部署情况
kubectl get pod -n argocd

kubectl edit service -n argocd argocd-server
# 将 Type 类型改成 NodePort，保存退出
# kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# 查看 NodePort
kubectl get service -n argocd argocd-server
#NAME            TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
# argocd-server   NodePort   10.0.0.6     <none>        80:31784/TCP,443:31830/TCP   24d
图中显示 NodePort 为31784，我们就可以使用 http://192.168.3.125:31784 进行访问，

# 登录需要用户名和密码，默认的用户名是 admin，密码保存在 argocd-initial-admin-secret 中，可以使用以下命令查看：
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo



```



```shell
在 UI 界面没有修改密码的地方，如果要修改密码，只能通过客户端操作。

可以在 https://github.com/argoproj/argo-cd/releases 页面选择合适的客户端，如果网络比较好，可以直接使用下面命令进行安装：

wget https://github.com/argoproj/argo-cd/releases/download/v2.4.9/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
copy
如果网络条件不好，可以使用以下命令进行安装：

wget https://gitee.com/coolops/go-hello-world/releases/download/v2.4.9/argocd-linux-amd64.zip
unzip argocd-linux-amd64.zip
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
copy
安装完成后使用 argocd version 查看信息，

发现查询 Argocd-Server 失败，这是因为我们需要登录，使用 argocd login 10.111.127.141:30888 进行登录，如下：
```

现在，我们可以使用 `argocd account update-password --account admin --current-password xxxx --new-password xxxx` 来修改密码，比如我这里修改成 `Argocd@123456`，完整命令如下：

```bash
argocd account update-password --account admin --current-password 04Y0P54LzoxGEYno --new-password Argocd@123456
```

修改成功输出如下：



# 配置 Webhook 加速 Argocd 配置更新

Argocd 会自动去拉配置进行更新，是定时每 3 分钟拉取一次.

为了消除轮询带来的延迟，可以将 API 服务器配置为接收 Webhook 事件。Argo CD 支持来自 GitHub，GitLab，Bitbucket，Bitbucket Server 和 Gogs 的 Git Webhook 通知，我们这里主要是配置 Gitlab。

#### 修改 argocd-secret，增加 Gitlab webhook token

使用 `kubectl edit secret argocd-secret -n argocd` 打开配置文件，新增以下代码：

```yaml
# 在顶级，和apiVersion、data 字段平级
stringData:
  webhook.gitlab.secret: test-argocd-token
stringData:
  webhook.gitlab.secret: cpy-token
```

点击保存过后保存到 argocd-secret 中，使用 `kubectl describe secret argocd-secret -n argocd` 查看，如下：

Gitlab webhook 配置:

然后在 Gitlab 上选择 `settings` -> `integrations`，配置 Webhook，如下：

```
URL:http://ip:port/api/webhook
Secret Token: test-argocd-token
# 由于集群内部证书是无效证书，所有要把 Enabled SSL 去掉
```

