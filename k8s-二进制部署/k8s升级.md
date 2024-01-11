k8s升级

https://jimmysong.io/kubernetes-handbook/practice/manually-upgrade.html



**下载地址:**  

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md

### Server Binaries

| filename                                                     | sha512 hash                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [kubernetes-server-linux-amd64.tar.gz](https://dl.k8s.io/v1.20.14/kubernetes-server-linux-amd64.tar.gz) | 0def92227a7770ff2792c3dcec5f3a6343792b98946171dc8e947c3adc9ecda2cee7aa8d695c5cb2fb5fcb5c82db8eb205b31fa42568b8aba010abbc25da2d0b |

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.22.md#downloads-for-v12217



### 升级master节点

停止master节点的进程

```bash
systemctl stop kube-apiserver
systemctl stop kube-scheduler
systemctl stop kube-controller-manager
systemctl stop kube-proxy
systemctl stop kubelet
```

使用新版本的kubernetes二进制文件替换原来老版本的文件，然后启动master节点上的进程：

```shell
tar zxvf kubernetes-server-linux-amd64.tar.gz
ips=(192.168.3.125 192.168.3.129)
for ip in ${ips[@]}
do
  ssh root@$ip systemctl stop kube-apiserver kube-scheduler kube-controller-manager
  scp kubernetes/server/bin/{kube-apiserver,kube-scheduler,kube-controller-manager} root@$ip:/opt/kubernetes/bin/
  scp kubernetes/server/bin/kubectl root@$ip:/usr/bin/
  ssh root@$ip systemctl start kube-apiserver kube-scheduler kube-controller-manager
  ssh root@$ip systemctl status kube-apiserver kube-scheduler kube-controller-manager
done
```

因为我们的master节点同时也作为node节点，所有还要执行下面的”升级node节点“中的步骤。

### 升级node节点

停止node节点上的kubernetes进程：

```bash
ips=(192.168.3.126 192.168.3.127 192.168.3.129)
for ip in ${ips[@]}
do
  ssh root@$ip systemctl stop kubelet kube-proxy
  scp kubernetes/server/bin/{kubelet,kube-proxy} root@$ip:/opt/kubernetes/bin/
  scp kubernetes/server/bin/kubectl root@$ip:/usr/bin/
  ssh root@$ip systemctl start kubelet kube-proxy
done
```

使用新版本的kubernetes二进制文件替换原来老版本的文件，然后启动node节点上的进程：

```bash
systemctl start kubelet
systemctl start kube-proxy
```



# 升级node和calico

### 一、升级Node

#### 1.1、升级kubelet（先升级master02）

```shell
# 1、下线k8s-master1
[root@k8s-master01 ~]# kubectl drain k8s-master1 --delete-emptydir-data --force --ignore-daemonsets

# 2、查看状态master02节点已经变成SchedulingDisabled状态
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS                     ROLES    AGE    VERSION
k8s-master01   Ready                      matser   5d5h   v1.19.5
k8s-master02   Ready,SchedulingDisabled   <none>   5d5h   v1.19.5
k8s-master03   Ready                      <none>   5d5h   v1.19.5
k8s-node01     Ready                      <none>   5d5h   v1.19.5
k8s-node02     Ready                      <none>   5d5h   v1.19.5

# 3、停止kubelet（master02节点上、升级那个节点就去那个节点停止，然后拷贝）
[root@k8s-master02 ~]# systemctl stop kubelet  

# 4、备份原文件且copy kubelet文件
[root@k8s-master02 ~]# cd /usr/local/bin/
[root@k8s-master02 bin]# mkdir kube-back
[root@k8s-master02 bin]# mv kubelet kube-back/
[root@k8s-master02 bin]# scp k8s-master01:/root/kubernetes/server/bin/kubelet .
[root@k8s-master02 bin]# ./kubelet --version    # 验证版本是否正确
Kubernetes v1.20.0


# 5、启动kubelet文件
！！！先不启动，先升级了Calico！！！
！！！请先往下走，升级了Calico，再启动kubelet！！！
```

### 二、升级Calico

#### 2.1、calico安装文档

```shell
# 安装方式有几种、参考官方文档选择吧
https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises
```

#### 2.2、官网升级文档

```shell
https://docs.projectcalico.org/maintenance/kubernetes-upgrade#upgrading-an-installation-that-uses-the-kubernetes-api-datastore
```

#### 2.3、下载

```shell
# 1、网络和网络策略管理都是用Calico
[root@k8s-master01 ~]# curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml -O    # 我用的是这个

# 2、网络策略管理用Calico和网络用flannel
curl https://docs.projectcalico.org/manifests/canal.yaml -O
```

#### 2.4、备份原来的cm、deploy

```shell
[root@k8s-master01 ~]# kubectl get cm -n kube-system  calico-config -oyaml > calico-config.yaml
[root@k8s-master01 ~]# kubectl get deploy -n kube-system  calico-kube-controllers -oyaml > calico-kube-controllers.yaml
```

#### 2.5、升级k8s-master02

```shell
# 1、查看版本。现在的版本（最新的）
[root@k8s-master01 ~]# cat calico.yaml | grep image    
          image: docker.io/calico/cni:v3.17.1
          image: docker.io/calico/cni:v3.17.1
          image: docker.io/calico/pod2daemon-flexvol:v3.17.1
          image: docker.io/calico/node:v3.17.1
          image: docker.io/calico/kube-controllers:v3.17.1
          
# 2、更改一下更新策略（防止更新失败）
[root@k8s-master01 ~]# vim calico.yaml 
# 原来的策略：
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
# 更改为：
  updateStrategy:
    type: OnDelete
# 然后apply它
[root@k8s-master01 ~]# kubectl apply -f calico.yaml 

# 3、查看是否加载成功
[root@k8s-master01 ~]# kubectl edit ds -n kube-system
# 查找calico的镜像，看到3.17.1说明更新成功
image: docker.io/calico/cni:v3.17.1
# 更新策略也变了
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: OnDelete
    
# 4、启动kubelet（在刚刚停止的节点启动，这次是升级的matser02，所以在02节点启动）
[root@k8s-master02 ~]# systemctl start kubelet

# 5、上线节点k8s-master02
[root@k8s-master01 ~]# kubectl uncordon k8s-master02
node/k8s-master02 uncordoned

# 6、查看版本是否正确
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS     ROLES    AGE    VERSION
k8s-master01   Ready      matser   5d6h   v1.19.5
k8s-master02   Ready      <none>   5d6h   v1.20.0   # 已经成功变成了v1.20.0
k8s-master03   Ready      <none>   5d6h   v1.19.5
k8s-node01     Ready      <none>   5d6h   v1.19.5
k8s-node02     NotReady   <none>   5d6h   v1.19.5

# 7、更新Calico  #这次更新的Calico是k8s-master02上的，所以只滚动更新k8s-master02 节点上的
[root@k8s-master01 ~]#  kubectl get pod -n kube-system -owide | grep k8s-master02 
calico-node-wz2l9       1/1     Running   9          5d6h   192.168.1.202   k8s-master02 
[root@k8s-master01 ~]#  kubectl delete pod -n kube-system  calico-node-wz2l9 
pod "calico-node-wz2l9" deleted

# 8、查看更新状态
[root@k8s-master01 ~]#  kubectl get pod -n kube-system -owide | grep k8s-master02 
calico-node-dr6pk       0/1     Init:0/3   0          19s    192.168.1.202   k8s-master02 

# 9、等状态变成Running，查看版本是否更新成功
[root@k8s-master01 ~]# kubectl edit po calico-node-dr6pk   -n kube-system

# 到此为止，一个节点完整的步骤已经走完了！那么我们更新一下其他的节点的吧
```

### 三、更新k8s-master01节点