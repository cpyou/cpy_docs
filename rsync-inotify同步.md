rsync-inotify同步



vim /etc/rsyncd.conf

```ini
# rsync 守护进程的用户
# uid = www
# 运行 rsync 守护进程的组
# gid = www
# 允许 chroot，提升安全性，客户端连接模块，首先 chroot 到模块 path 参数指定的目录下，chroot 为 yes 时必须使用 root 权限，且不能备份 path 路径外的链接文件
use chroot = yes
# 只读
read only = no
# 只写
write only = no
# 设定白名单，可以指定IP段（172.18.50.1/255.255.255.0）,各个Ip段用空格分开
hosts allow = 10.0.1.54
hosts deny = *
# 允许的客户端最大连接数
max connections = 4
# 欢迎文件的路径，非必须
motd file = /etc/rsyncd.motd
# pid文件路径
pid file = /var/run/rsyncd.pid
# 记录传输文件日志
transfer logging = yes
# 日志文件格式
log format = %t %a %m %f %b
# 指定日志文件
log file = /var/log/rsync.log
# 剔除某些文件或目录，不同步
exclude = lost+found/
# 设置超时时间
timeout = 900
ignore nonreadable = yes
# 设置不需要压缩的文件
dont compress   = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2

# 模块，可以配置多个
[sync_file]
# 模块的根目录，同步目录，要注意权限
path = /home/ubuntu/cpy/sync-dir
# 是否允许列出模块内容
list = no
# 忽略错误
ignore errors
# 添加注释
comment = ftp export area
# 模块验证的用户名称，可使用空格或者逗号隔开多个用户名
auth users = ubuntu
# 模块验证密码文件 可放在全局配置里
# secrets file = /etc/rsyncd.secrets
```

```shell
rsync -e "ssh" -avzP --delete ubuntu@10.0.1.39::sync_file /home/ubuntu/cpy/sync-dir
rsync -avzP --delete /home/ubuntu/cpy/sync-dir sync@10.0.1.39::sync_file --password-file=/etc/rsyncd.pass
rsync -e "ssh" -avzP --delete  /home/ubuntu/cpy/sync-dir/ ubuntu@10.0.1.54:/home/ubuntu/cpy/sync-dir/

rsync --daemon --config=/etc/rsyncd.conf
sudo kill $(cat /var/run/rsyncd.pid)
```



```
sudo apt  install -y inotify-tools
```

```shell
# vim monitor.sh
#!/bin/bash

Path=/home/ubuntu/cpy/sync-dir/
Server=10.0.1.54
User=ubuntu
module=sync_file

monitor() {
  /usr/bin/inotifywait -mrq --format '%w%f' -e create,close_write,delete $1 | while read line; do
    if [ -f $line ]; then
      rsync -avz $line --delete ${User}@${Server}:${Path} -e "ssh"
    else
      cd $1 &&
        rsync -avz ./ --delete ${User}@${Server}:${Path} -e "ssh"
    fi
  done
}

monitor $Path;
```



