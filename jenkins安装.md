# 一、概述

Jenkins 是一个基于 Java 语言开发的持续构建工具平台，主要用于持续、自动的构建/测试你的软件和项目。它可以执行你预先设定好的设置和构建脚本，也可以和 Git 代码库做集成，实现自动触发和定时触发构建。

# 二、安装

1、安装 OpenJDK

```shell
yum install -y fontconfig java-11-openjdk
java --version

```

2、安装 Jenkins

```shell
# 使用 wget 下载 Jenkins 软件包的存储库配置文件，并将其保存到 /etc/yum.repos.d/jenkins.repo 文件中
# 证书过期，不检查证书：sudo wget --no-check-certificate -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
# 使用 rpm 导入 Jenkins 软件包的 GPG 密钥，以确保安装的软件包是经过验证的，并且没有被篡改过
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
# 安装 EPEL 的发行包，通过安装 EPEL 发行包，您可以访问一些常用的第三方软件包
yum install -y epel-release
# 使用 yum 安装 Jenkins 软件包
yum install -y jenkins
```

3、启动 Jenkins

```shell
vim /usr/lib/systemd/system/jenkins.service

User=root
Group=root

systemctl daemon-reload
systemctl start jenkins.service
systemctl stop jenkins.service
systemctl status jenkins.service
```

4、给 Jenkins 放行端口
在启动 Jenkins 后，此时 Jenkins 会开启它的默认端口 8080 。但由于防火墙限制，我们需要手动让防火墙放行 8080 端口才能对外访问到界面。

这里我们在 CentOS 下的 firewall-cmd 防火墙添加端口放行规则，添加完后重启防火墙。

```shell
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --zone=public --add-port=50000/tcp --permanent

systemctl reload firewalld
```

# 三、初始化 Jenkins 配置

1、访问