# 打通Kubernetes内网与局域网的N种方法

https://zhuanlan.zhihu.com/p/187548589

```shell
# k8s-二进制部署中可以按一下设置直接访问
# 命令中下面两处需要替换成真实的值
# 集群子网： 10.0.0.0 /24 or 255.255.255.0
# 集群某节点： 192.168.3.25

# Windows 
route ADD 10.0.0.0 MASK 255.240.0.0 192.168.3.25

# Linux
sudo ip route add 10.0.0.0/24 via 192.168.3.25 dev eth0

# MacOS
sudo route -n add -net 10.0.0.0 -netmask 255.255.255.0 192.168.3.25
sudo route -v delete -net 10.0.0.0 -netmask 255.255.255.0 192.168.3.25 # 删除路由
 

# 如果在网关上（路由器/交换机）统一配置时，各个厂商设备的命令有所不同

 kubectl get pods,svc -n kubernetes-dashboard
 NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-66dd8bdd86-hnffr   1/1     Running   7          54d
pod/kubernetes-dashboard-76b8456c86-kw555        1/1     Running   4          47d

NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.0.0.140   <none>        8000/TCP        54d
service/kubernetes-dashboard        NodePort    10.0.0.185   <none>        443:31919/TCP   54d
# 可直接用https://10.0.0.185 访问kubernetes-dashboard
```

