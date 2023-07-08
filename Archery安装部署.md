Archery 安装部署





```shell
# 安装依赖
sudo yum install -y libffi-devel wget gcc make zlib-devel openssl openssl-devel ncurses-devel openldap-devel gettext bzip2-devel xz-devel

pyenv install 3.9.10
pyenv local 3.9.10
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# pip config set global.index-url https://mirrors.ustc.edu.cn/pypi/web/simple/ 
pip install virtualenvwrapper 
echo "export VIRTUALENVWRAPPER_PYTHON=/root/.pyenv/versions/3.9.10/bin/python" >> ~/.bashrc
echo "export VIRTUALENVWRAPPER_VIRTUALENV=/root/.pyenv/versions/3.9.10/bin/virtualenv" >> ~/.bashrc
echo "source /root/.pyenv/versions/3.9.10/bin/virtualenvwrapper.sh" >> ~/.bashrc
source ~/.bashrc

mkvirtualenv -p ~/.pyenv/versions/3.9.10/bin/python archery
git clone https://github.com/hhyo/Archery.git
cd Archery
pip3 install -r requirements.txt

# OSError: mysql_config not found
# yum install -y mysql-devel gcc gcc-devel python3-devel

# error: command '/usr/bin/gcc' failed with exit code 1  ERROR: Failed building wheel for pyodbc 
sudo yum install -y python3-devel gcc-c++ unixODBC unixODBC-devel
# error: command '/usr/bin/gcc' failed with exit code 1  ERROR: Failed building wheel for python-ldap
sudo yum -y install openldap-devel

```

### 安装Inception(MySQL审核、查询校验和数据脱敏)

```shell
# 安装Inception(MySQL审核、查询校验和数据脱敏)
git clone https://github.com/hanchuanchuan/goInception.git
cd goInception
go build -o goInception tidb-server/main.go

cp config/config.toml.default config/config.toml
./goInception -config=config/config.toml

# verifying github.com/coreos/etcd@v3.3.10+incompatible: checksum mismatch
go clean -modcache
cd goInception
rm go.sum
go mod tidy
go build -o goInception tidb-server/main.go
```

### 安装redis

```shell
sudo yum install -y redis
sudo vim /etc/redis/redis.conf
# 输入/requirepass 找到requirepass关键字，后面跟的就是密码，默认是注释掉的，即不需要密码
# 注释打开，后面修改为自己的密码
requirepass 123456

sudo systemctl restart redis
sudo systemctl enable redis
```





### 启动准备

```shell
# 安装mariadb
sudo yum install -y wget
wget https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
chmod +x mariadb_repo_setup
./mariadb_repo_setup
sudo yum install -y MariaDB-server
sudo systemctl start mariadb
sudo systemctl status mariadb
sudo systemctl enable mariadb

# 创建数据库
mysql -uroot
create database archery;
show variables like '%char%';
ALTER DATABASE archery DEFAULT CHARACTER SET utf8mb4;

# 修改配置
vi archery/settings.py

# 数据库初始化
python3 manage.py makemigrations sql
python3 manage.py migrate 

# 数据初始化
python3 manage.py dbshell < sql/fixtures/auth_group.sql
python3 manage.py dbshell < src/init_sql/mysql_slow_query_review.sql

# 创建管理用户
python3 manage.py createsuperuser
```

## 启动

### runserver启动（仅作为本地测试）

```shell
source /opt/venv4archery/bin/activate
#启动Django-Q，需保持后台运行
python3 manage.py qcluster
#启动服务
python3 manage.py runserver 0.0.0.0:9123  --insecure   
```

### Gunicorn+Nginx启动

```shell
# nginx配置示例  
server{
    listen 9123; # 监听的端口
    server_name archery;
    client_max_body_size 20M; # 处理Request Entity Too Large
    proxy_read_timeout 600s;  # 超时时间与Gunicorn超时时间设置一致，主要用于在线查询

    location / {
      proxy_pass http://127.0.0.1:8888;
      proxy_set_header Host $host:9123; # 解决重定向404的问题，和listen端口保持一致，如果是docker则和宿主机映射端口保持一致
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static {
      alias /opt/archery/static; # 此处指向settings.py配置项STATIC_ROOT目录的绝对路径，用于nginx收集静态资源
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
} 
# 启动  

source /opt/venv4archery/bin/activate
bash startup.sh
mkdir -p /opt/archery/static/
sudo cp -rf static/* /opt/archery/static/
# 使用ssl 或者openresty 需要配置CSRF_TRUSTED_ORIGINS 否则访问会报Forbiden

```

## 访问

http://127.0.0.1:9123/



防火墙开启

```shell
# 外部无法访问
sudo firewall-cmd --zone=public --add-port=9123/tcp --permanent

# connect() to 127.0.0.1:8888 failed (13: Permission denied) while connecting to upstream
sudo yum install -y policycoreutils-python
sudo semanage port -a -t http_port_t -p tcp 8888
sudo semanage port --list | grep 8888

```





脱密规则数据

```sql
INSERT INTO `data_masking_rules` (`id`,`rule_type`,`rule_regex`,`hide_group`,`rule_desc`,`sys_time`) 
VALUES 
  ('1','1','(.{3})(.*)(.{4})','2','保留前三后四','2023-05-29 21:22:54.933130'),
  ('2','2','(.{6})(.*)(.{4})','2','保留前6后4','2023-05-29 21:24:38.358667'),
  ('3','3','(.{6})(.*)(.{4})','2','保留前6后4','2023-05-29 21:24:58.257928'),
  ('4','4','(.{1})(.*)(@.*)','2','去除后缀','2023-05-29 21:26:21.195815'),
  ('5','5','(.*)(.{4})$','2','隐藏后四位','2023-05-29 21:28:42.944027');
# 依次是：手机号、证件号码、银行卡、邮箱、金额
```

