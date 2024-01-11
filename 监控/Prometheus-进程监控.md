# Prometheus-进程监控

```shell
wget https://github.com/ncabatoff/process-exporter/releases/download/v0.7.10/process-exporter-0.7.10.linux-amd64.tar.gz 

mkdir /usr/local/process-exporter && tar -zxvf process-exporter-0.7.10.linux-amd64.tar.gz -C /usr/local/process-exporter --strip-components 1

cat > /usr/lib/systemd/system/process-exporter.service << EOF 
[Unit] 
Description=process-exporter 
Documentation=https://github.com/ncabatoff/process-exporter 
After=network.target 

[Service] 
Type=simple 
ExecStart=/usr/local/process-exporter/process-exporter -config.path=/usr/local/process-exporter/process-conf.yaml 
Restart=always 

[Install] 
WantedBy=multi-user.target

EOF

systemctl daemon-reload 
systemctl enable process-exporter 
systemctl start process-exporter 
systemctl status process-exporter
```



process-conf.yaml 配置： 进入/usr/local/process-exporter/ 

```yaml
# vi process-conf.yaml
process_names:
  - name: "{{.Matches}}"
cmdline:
  - 'dc-live'
  - name: "{{.Matches}}"
cmdline:
  - 'prism'
  - name: "{{.Matches}}"
cmdline:
  - '/opt/atlassian/confluence/bin/tomcat-juli.jar'
```



然后进入Jenkins注册服务：
