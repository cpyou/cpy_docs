

# Centos7 安装sonar9.4



------

### 1. **系统准备**

确保系统已更新并安装必要的依赖。

```shell
sudo yum update -y
sudo yum install -y wget unzip java-11-openjdk-devel
```

验证 Java 安装：

```shell
java -version
```

------

### 2. **安装 PostgreSQL 数据库**

SonarQube 需要数据库支持，这里以 PostgreSQL 为例。

1. 安装 PostgreSQL：

   ```
   sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   sudo yum install -y postgresql12-server postgresql12-contrib
   ```

2. 初始化数据库并启动服务：

    ```shell
    sudo /usr/pgsql-12/bin/postgresql-12-setup initdb
    sudo systemctl start postgresql-12
    sudo systemctl enable postgresql-12
    ```

3. 创建 SonarQube 数据库和用户：

    ```shell
    sudo -u postgres psql
    CREATE DATABASE sonarqube;
    CREATE USER sonarqube WITH PASSWORD 'sonarqube';
    GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonarqube;
    \q
	```

4. 修改 PostgreSQL 配置文件以允许远程连接（可选）：

编辑 `/var/lib/pgsql/12/data/pg_hba.conf`，添加以下行：

    ```properties
    host    all             all             0.0.0.0/0               md5
    # 上述配置有重复的需要注释，
    ```

编辑 `/var/lib/pgsql/12/data/postgresql.conf`，修改以下行：

```properties
listen_addresses = '*'
```

重启 PostgreSQL 服务：

```shell
sudo systemctl restart postgresql-12
```

------

### 3. **下载并安装 SonarQube**

1. 下载 SonarQube：

    ```shell
    cd /opt
    sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
    sudo unzip sonarqube-9.4.0.54424.zip
    sudo mv sonarqube-9.4.0.54424 sonarqube
    ```

2. 创建 SonarQube 用户并设置权限：

    ```shell
    sudo useradd -M -d /opt/sonarqube -s /bin/bash -r sonarqube
    sudo chown -R sonarqube:sonarqube /opt/sonarqube
    ```

------

### 4. **配置 SonarQube**

1. 编辑 `sonar.properties` 文件：

    ```shell
    sudo vim /opt/sonarqube/conf/sonar.properties
    ```

修改以下配置：

    ```properties
    sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
    sonar.jdbc.username=sonarqube
    sonar.jdbc.password=sonarqube
    sonar.web.host=0.0.0.0
    sonar.web.port=9000
    ```

2. 编辑 `sonar.sh` 文件以设置运行用户：

    ```shell
    sudo vim /opt/sonarqube/bin/linux-x86-64/sonar.sh
    ```

		找到以下行并修改为：

    ```properties
    RUN_AS_USER=sonarqube
    ```

------

### 5. **启动 SonarQube**

1. 启动 SonarQube：

```shell
sudo /opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

2. 查看日志以确认启动状态：

```shell
tail -f /opt/sonarqube/logs/sonar.log
```

3. 访问 SonarQube：
   打开浏览器，访问 `http://<服务器IP>:9000`，默认用户名和密码为 `admin`。

------

### 6. **安装 SonarScanner**

SonarScanner 用于分析代码并上传结果到 SonarQube。

1. 下载并安装 SonarScanner：

```shell
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.6.2.2472-linux.zip
sudo unzip sonar-scanner-cli-4.6.2.2472-linux.zip
sudo mv sonar-scanner-4.6.2.2472-linux sonar-scanner
```

2. 配置环境变量：

```shell
echo 'export PATH=$PATH:/opt/sonar-scanner/bin' | sudo tee -a /etc/profile
source /etc/profile
```

3. 验证安装：

```shell
sonar-scanner -v
```

------

### 7. **使用 SonarScanner 分析项目**

1. 在项目根目录创建 `sonar-project.properties` 文件：

```properties
sonar.projectKey=myproject
sonar.projectName=My Project
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
```

2. 运行 SonarScanner：

```
sonar-scanner
```

------

### 8. **配置为系统服务（可选）**

1. 创建 systemd 服务文件：

```shell
sudo vim /etc/systemd/system/sonarqube.service
```

内容如下：

```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonarqube
Group=sonarqube
Restart=always
# LimitNOFILE、LimitNPROC不配置ES无法启动
LimitNOFILE=65536
LimitNPROC=4096 

[Install]
WantedBy=multi-user.target
```

2. 启动并启用服务：

```shell
sudo systemctl start sonarqube
sudo systemctl enable sonarqube
```

------

### 总结

以上是在 CentOS 7 上安装和配置 SonarQube 的完整步骤。安装完成后，你可以通过 SonarQube 持续监控代码质量。