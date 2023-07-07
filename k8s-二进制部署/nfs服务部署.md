现在有3台[服务器](https://cloud.tencent.com/product/cvm?from=10680) s1(主)，s2(从), s3（从）需要实现文件实时同步，我们可以安装Nfs服务端和客户端来实现！ 

|          |              |
| -------- | ------------ |
| s1(主)   | 192.168.3.25 |
| s2(从)   | 192.168.3.26 |
| s3（从） | 192.168.3.27 |

**一、安装 NFS 服务器所需的软件包：** 

```shell
yum install -y nfs-utils

# Ubuntu 安装
sudo apt-get -y install nfs-kernel-server
```

复制

**二、编辑exports文件，添加从机**

```shell
mkdir /home/nfs
chmod 755 /home/nfs
vim /etc/exports
/home/nfs/ 192.168.3.0/24(rw,sync,no_root_squash,no_all_squash)
/home/ubuntu/cpy/nfs/ 10.0.1.54(rw,sync,no_root_squash,no_all_squash)
```

1. `/home/nfs/`: 共享目录位置。
2. `192.168.0.0/24`: 客户端 IP 范围，`*` 代表所有，即没有限制。
3. `rw`: 权限设置，可读可写。
4. `sync`: 同步共享目录。
5. `no_root_squash`: 可以使用 root 授权。
6. `no_all_squash`: 可以使用普通用户授权。

**三、启动nfs服务**

先为rpcbind和nfs做开机启动：(必须先启动rpcbind服务)

```shell
systemctl enable rpcbind.service
systemctl enable nfs-server.service
```

然后分别启动rpcbind和nfs服务：

```shell
systemctl start rpcbind.service
systemctl start nfs-server.service
```

确认NFS服务器启动成功：

```shell
rpcinfo -p
showmount -e localhost
```

检查 NFS 服务器是否挂载我们想共享的目录 /home/nfs/： 

```shell
exportfs -r
```

\#使配置生效 

```shell
exportfs
```

 \#可以查看到已经ok 

```shell
/home/nfs 192.168.3.0/24
```

复制

**四、在从机上安装NFS 客户端**

首先是安裝nfs，同上，然后启动rpcbind服务

```shell
yum install -y nfs-utils

# Ubuntu 安装
sudo apt-get -y install nfs-common
```

先为rpcbind做开机启动 然后启动rpcbind服务： 

```shell
systemctl enable rpcbind.service
systemctl start rpcbind.service
```

注意：客户端不需要启动nfs服务

检查 NFS 服务器端是否有目录共享：showmount -e nfs服务器的IP 

```javascript
showmount -e 192.168.3.25
Export list for 192.168.248.208:
/home/nfs 192.168.248.0/24

```

在从机上使用 mount 挂载服务器端的目录/home/nfs到客户端某个目录下： 

```javascript
mkdir -p /home/nfs
mount -t nfs 192.168.3.25:/home/nfs /home/nfs
```

查看是否挂载成功

```
df -h 
mount
```

# 五.nfs服务端使用固定端口并启用防火墙开放

> nfs通信是使用udp或tcp协议进行的，上面的nfs环境是建立在nfs服务器防火墙关闭的情况下的搭建，`即需要已经放通相关的端口`，一般线上环境要求较高，会开启防火墙并授权一些策略来控制访问，由于nfs默认除了用`111`（portmapper使用，客户端向服务器发出NFS文件存取功能的询问请求）和`2049`（nfs使用）端口是固定的，其他的几个端口是随机的，因此需要在nfs服务器(192.168.3.25)上配置成固定的端口，再通过防火墙进行开放，步骤如下：

1. 修改配置文件 vi /etc/sysconfig/nfs
```shell
vi /etc/sysconfig/nfs # 在最后加上以下配置
RQUOTAD_PORT=30001
LOCKD_TCPPORT=30002
LOCKD_UDPPORT=30002
MOUNTD_PORT=30003
STATD_PORT=30004
```

2. 重启服务

```shell
systemctl restart rpcbind
systemctl restart nfs
```

3. 重新查看端口情况 rpcinfo -p localhost，可看到端口已经使用固定的端口 

```shell
rpcinfo -p localhost
```

>    program vers proto   port  service
>     100000    4   tcp    111  portmapper
>     100000    3   tcp    111  portmapper
>     100000    2   tcp    111  portmapper
>     100000    4   udp    111  portmapper
>     100000    3   udp    111  portmapper
>     100000    2   udp    111  portmapper
>     100005    1   udp  30003  mountd
>     100005    2   udp  30003  mountd
>     100005    3   udp  30003  mountd
>     100003    3   tcp   2049  nfs
>     100003    4   tcp   2049  nfs
>     100227    3   tcp   2049  nfs_acl
>     100003    3   udp   2049  nfs
>     100003    4   udp   2049  nfs
>     100227    3   udp   2049  nfs_acl
>     100021    1   udp  30002  nlockmgr
>     100021    3   udp  30002  nlockmgr
>     100021    4   udp  30002  nlockmgr
>     100021    1   tcp  30002  nlockmgr
>     100021    3   tcp  30002  nlockmgr
>     100021    4   tcp  30002  nlockmgr

3. 接下来进行相关的防火墙开放，授权在客户端机器192.168.3.26上能访问

```shell
#nfs之防火墙
nfs_client_ip=192.168.3.26
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$nfs_client_ip" port protocol="tcp" port="111" accept"
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$nfs_client_ip" port protocol="udp" port="111" accept"
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$nfs_client_ip" port protocol="tcp" port="2049" accept"
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$nfs_client_ip" port protocol="udp" port="2049" accept"
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$nfs_client_ip" port protocol="tcp" port="30001-30004" accept"
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$nfs_client_ip" port protocol="udp" port="30001-30004" accept"
firewall-cmd --reload
firewall-cmd --list-all
```

### 测试 NFS

测试一下，在客户端向共享目录创建一个文件

```
cd /home/nfs
sudo touch a
```

之后取 NFS 服务端 `192.168.3.25` 查看一下

```
$ cd /home/nfs
$ ll
total 0
-rw-r--r--. 1 root root 0 Aug  8 18:46 a
```

### 客户端自动挂载

自动挂载很常用，客户端设置一下即可。

```shell
vi /etc/fstab

192.168.3.161:/home/nfs /home/nfs                  nfs     defaults        0 0
```

```shell
systemctl daemon-reload
```