nacos安装

https://nacos.io/docs/latest/guide/admin/cluster-mode-quick-start/

centos 7 

下载nacos：https://github.com/alibaba/nacos/releases

wget https://github.com/alibaba/nacos/releases/download/2.3.2/nacos-server-2.3.2.tar.gz



```shell
yum install -y java-11-openjdk

which java
# /usr/bin/java

ls -lrt /usr/bin/java
# /usr/bin/java -> /etc/alternatives/java

ls -lrt /etc/alternatives/java
# /etc/alternatives/java -> /usr/lib/jvm/java-11-openjdk-11.0.22.0.7-1.el7_9.x86_64/bin/java

cat >> /etc/profile << EOF 
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.22.0.7-1.el7_9.x86_64
EOF
source /etc/profile
```



# 单机模式启动调试

```
cd nacos/bin
sh startup.sh -m standalone

```

## 开启防火墙

```shell
sudo firewall-cmd --zone=public --add-port=8848/tcp --permanent
sudo firewall-cmd --reload
```

# 集群安装

##  1、集群

  Nacos单击模式仅仅适用于测试和单击使用，生产环境大多使用集群模式以确保高可用。如果有多数据中心场景，那么Nacos还支持多集群模式。 nacos集群架构图如下： 

![在这里插入图片描述](https://developer.qcloudimg.com/http-save/yehe-7952434/6dbd5fd555fb09f14996c001baaf866b.jpg)

在这里插入图片描述

>  因此开源的时候推荐用户把所有服务列表放到一个vip下面，然后挂到一个[域名](https://cloud.tencent.com/act/pro/domain-sales?from_column=20065&from=20065)下面 http://ip1:port/openAPI 直连ip模式，机器挂则需要修改ip才可以使用。 http://SLB:port/openAPI 挂载SLB模式(内网SLB，不可暴露到公网，以免带来安全风险)，直连SLB即可，下面挂server真实ip，可读性不好。 http://nacos.com:port/openAPI 域名 + SLB模式(内网SLB，不可暴露到公网，以免带来安全风险)，可读性好，而且换ip方便，推荐模式

## 2、集群搭建注意事项

- 3个或3个以上nacos节点才能构成集群。
- 要求虚拟机内存分配必须大于3G以上。
- 数据持久化必须配置为mysql持久化

## 3、集群规划

```
node cluster:
 192.168.3.180 8845 nacos01
 192.168.3.180 8846 nacos02
 192.168.3.180 8847 nacos03
 192.168.3.180 9090 nginx
 192.168.3.180 3306 mysql
```

## 4、搭建nacos集群

### 4.1 准备3个nacos节点，并连接mysql数据库

将nacos安装包复制三份：nacos01、nacos02、nacos03

```shell
cp -rf nacos nacos01
cp -rf nacos nacos02
cp -rf nacos nacos03
```



### 4.2 重新初始化mysql数据

#### 初始化 MySQL 数据库

[sql语句源文件](https://github.com/alibaba/nacos/blob/master/distribution/conf/mysql-schema.sql)



### 4.3 修改nacos conf目录中cluster.conf文件添加所有集群节点

第一台：

```
cd nacos01
cp conf/cluster.conf.example conf/cluster.conf
# 添加集群节点，删除示例
cat >> conf/cluster.conf << EOF 
192.168.3.180:8845
192.168.3.180:8846
192.168.3.180:8847
EOF
vim conf/cluster.conf

```

### 4.4 修改nacos各自端口号

```shell
sed -i 's|server.port=8848|server.port=8845|g' nacos01/conf/application.properties
sed -i 's|server.port=8848|server.port=8846|g' nacos02/conf/application.properties
sed -i 's|server.port=8848|server.port=8847|g' nacos03/conf/application.properties

# 检查
cat nacos01/conf/application.properties | grep server.port
cat nacos02/conf/application.properties | grep server.port
cat nacos03/conf/application.properties | grep server.port
```

### 4.5 启动三台nacos节点

```
# 内置数据源启动
./nacos01/bin/startup.sh -p embedded
./nacos02/bin/startup.sh -p embedded
./nacos03/bin/startup.sh -p embedded

sh ./nacos01/bin/shutdown.sh
sh ./nacos02/bin/shutdown.sh
sh ./nacos03/bin/shutdown.sh
```

### 4.6 测试集群是否搭建成功

  在微服务中向8845端口注册，若其他两个nacos节点也注册了该服务，则证明集群搭建成功

启动服务之后，查看三台节点： 192.168.3.180:8845

```
sudo firewall-cmd --zone=public --add-port=8845/tcp --permanent
sudo firewall-cmd --reload
```

