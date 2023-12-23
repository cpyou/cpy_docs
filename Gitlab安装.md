Gitlab安装



#### 2.yum安装

#### 新建 /etc/yum.repos.d/gitlab-ce.repo，内容为：

```ini
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
gpgcheck=0
enabled=1
```

#### 安装依赖

```shell
sudo yum install -y curl openssh-server openssh-clients postfix cronie
sudo systemctl start postfix 
sudo systemctl enable postfix 
sudo chkconfig postfix on
```

#### 安装gitlab-ce

```shell
sudo yum makecache
sudo yum install -y gitlab-ce
sudo gitlab-ctl reconfigure  #Configure and start GitLab
sudo vim /etc/gitlab/gitlab.rb        # 修改默认的配置文件；
# external_url 'http://192.168.3.172:8081'


```

开启防火墙

```shell
sudo firewall-cmd --zone=public --add-port=8081/tcp --permanent
sudo firewall-cmd --reload
```

#### gitlab发送邮件配置

```ini
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "cpy@domain.com"
gitlab_rails['smtp_password'] = "smtp password"
gitlab_rails['smtp_authentication'] = "plain"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['gitlab_email_from'] = "cpy@domain.com"
gitlab_rails['gitlab_email_reply_to'] = "cpy@domain.com"
```

#### 服务器修改过ssh端口的坑(需要修改配置ssh端口)

```shell
#修改过ssh端口，gitlab中项目的的ssh地址，会在前面加上协议头和端口号“ssh://git@gitlab.domain.com:55725/huangdc/test.git”
gitlab_rails['gitlab_shell_ssh_port'] = 55725
```

**修改密码**

```
gitlab-rails console -e production
user = User.where(id:1).first
user.password='cpy12345'
user.password_confirmation = 'cpy12345'
user.save!
```



## GitLab备份和恢复

#### 备份

```shell
# 可以将此命令写入crontab，以实现定时备份
/usr/bin/gitlab-rake gitlab:backup:create
```

备份的数据会存储在/var/opt/gitlab/backups，用户通过自定义参数 gitlab_rails['backup_path']，改变默认值。

#### 恢复

```shell
# 停止unicorn和sidekiq，保证数据库没有新的连接，不会有写数据情况
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq

# 进入备份目录进行恢复，1476900742为备份文件的时间戳
cd /var/opt/gitlab/backups
gitlab-rake gitlab:backup:restore BACKUP=1476900742
cd -

# 启动unicorn和sidekiq
sudo gitlab-ctl start unicorn
sudo gitlab-ctl start sidekiq
```



参考：https://www.jianshu.com/p/b04356e014fa
