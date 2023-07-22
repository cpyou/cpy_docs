aws 使用s3作为共享目录

### 利用S3fs在Amazon EC2 Linux实例上挂载S3存储桶

```shell
https://aws.amazon.com/cn/blogs/china/s3fs-amazon-ec2-linux/

# 安装必要的软件包
# Ubuntu 系统
sudo apt install autoconf libxml2-dev libfuse-dev libcurl4-openssl-dev
sudo apt download autoconf libxml2-dev libfuse-dev libcurl4-openssl-dev m4 libcurl4 libxml2 libicu-dev libicu66 icu-devtools automake libtool autotools-dev
# centos
sudo yum install automake fuse fuse-devel gcc-c++ git libcurl-devel libxml2-devel make openssl-devel


# 下载，编译并安装s3fs
git clone https://github.com/s3fs-fuse/s3fs-fuse.git
cd s3fs-fuse
sudo ./autogen.sh
sudo ./configure
sudo make
sudo make install
# 检查s3fs是否安装成功
s3fs --help


sudo su odoo
# 创建IAM用户访问密钥文件
# echo [IAM用户访问密钥ID]:[ IAM用户访问密钥] >[密钥文件名]
echo AK.....:6TBr...... > /home/ec2-user/.passwd-s3fs
chmod 600 /home/ec2-user/.passwd-s3fs


# 手动挂载S3存储桶
mkdir /home/ec2-user/s3mnt
s3fs s3fs-mount-bucket /home/ec2-user/s3mnt -o passwd_file=/home/ec2-user/.passwd-s3fs -o endpoint=cn-northwest-1
# 查看
df -h
# 卸载
umount /home/ec2-user/s3mnt

aws s3 ls s3://catalog-dev
aws configure list
```

