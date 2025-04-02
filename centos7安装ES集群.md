```shell
三台机器执行: sudo systemctl stop firewalld

ssh root@192.168.3.181

rpm -i elasticsearch-8.17.4-x86_64.rpm
warning: elasticsearch-8.17.4-x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID d88e42b4: NOKEY
Creating elasticsearch group... OK
Creating elasticsearch user... OK
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : EeLR2Zlsj+YNuOaS8QAA

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service

systemctl daemon-reload
vim /etc/elasticsearch/elasticsearch.yml
修改：network.host: 0.0.0.0

systemctl start elasticsearch

export ELASTIC_PASSWORD="EeLR2Zlsj+YNuOaS8QAA"
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://127.0.0.1:9200
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://127.0.0.1:9200/_cat/nodes?v
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://127.0.0.1:9200/_cluster/health?pretty

/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node
# eyJ2ZXIiOiI4...

# 注意上述token会过期，生成后续立即使用。 root@192.168.3.182
/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>

# 主节点执行
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token --scope node
# eyJ2ZXIiO...

# ssh root@192.168.3.183
/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>


rpm -i kibana-8.17.4-x86_64.rpm

warning: kibana-8.17.4-x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID d88e42b4: NOKEY
Creating kibana group... OK
Creating kibana user... OK
Kibana is currently running with legacy OpenSSL providers enabled! For details and instructions on how to disable see https://www.elastic.co/guide/en/kibana/8.17/production.html#openssl-legacy-provider
Created Kibana keystore in /etc/kibana/kibana.keystore

vim /etc/kibana/kibana.yml
server.host: "0.0.0.0"
# 增加ES节点后添加其他IP
# elasticsearch.hosts: ['https://192.168.3.181:9200', 'https://192.168.3.182:9200', 'https://192.168.3.183:9200']
systemctl start kibana
journalctl -f -u kibana

根据启动日志浏览器访问：http://192.168.3.181:5601/?code=159824

/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
eyJ2ZXIiOiI4LjE0LjAiLCJhZHIiOlsiMTAuMTY4Ljk5LjM1OjkyMDAiXSwiZmdyIjoiZTY2ZTE2ZGM1OGFjNTU3YWNmZDQ3NzM2NzAyOTcwMTdiN2NjODViYzg5MTBmMzc1YTI5MzUyY2MxMDgzYjc2ZSIsImtleSI6Ik5xeHI5SlVCNVFZTE1YZGlfX0FNOmZ4QkNjalRtUmY2VnFYVER6dkZBM3cifQ==


# 重装elasticsearch
systemctl stop elasticsearch
rpm -e elasticsearch
rm -rf /var/lib/elasticsearch /etc/elasticsearch /var/log/elasticsearch
rpm -i elasticsearch-8.17.4-x86_64.rpm
```







