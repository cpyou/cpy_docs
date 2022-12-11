现在有3台[服务器](https://cloud.tencent.com/product/cvm?from=10680) s1(主)，s2(从), s3（从）需要实现文件实时同步，我们可以安装Nfs服务端和客户端来实现！ 

|          |              |
| -------- | ------------ |
| s1(主)   | 192.168.3.25 |
| s2(从)   | 192.168.3.26 |
| s3（从） | 192.168.3.27 |

**一、安装 NFS 服务器所需的软件包：** 

```javascript
yum install -y nfs-utils
```

复制

**二、编辑exports文件，添加从机**

```shell
mkdir /home/nfs
chmod 755 /home/nfs
vim /etc/exports
/home/nfs/ 192.168.3.0/24(rw,sync,no_root_squash,no_all_squash)
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

```
vi /etc/fstab

192.168.3.25:/home/nfs /home/nfs                   nfs     defaults        0 0
```

```shell
systemctl daemon-reload
```