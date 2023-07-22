Ubuntu 挂载EFS

www.baidu.com

```shell
$ sudo apt-get update
$ sudo apt-get -y install git binutils
$ git clone https://github.com/aws/efs-utils
$ cd efs-utils
$ ./build-deb.sh
$ sudo apt-get -y install ./build/amazon-efs-utils*deb

sudo mkdir /mnt/efs

sudo chown <用户名>:<用户组名> /mnt/efs
# 手动挂载
sudo mount -t efs file-system-id efs-mount-point/ 
sudo mount -t efs fs-abcd123... efs/   # 具体示例

# 配置自动挂载：如果希望在 EC2 实例重启后自动挂载 EFS 文件系统，可以将挂载命令添加到 `/etc/fstab` 文件中：
<文件系统的 DNS 名称>:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0

file_system_id.efs.aws-region.amazonaws.com:/ mount_point nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0
#查看挂载
sudo mount -a

```

