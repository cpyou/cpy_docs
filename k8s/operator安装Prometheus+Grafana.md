operator安装Prometheus+Grafana



```shell
cd /root; 
git clone -b release-0.10 https://github.com/prometheus-operator/kube-prometheus.git;
cd kube-prometheus/manifests;
sed -i "s#quay.io/prometheus/#registry.cn-hangzhou.aliyuncs.com/chenby/#g" *.yaml;
sed -i "s#quay.io/brancz/#registry.cn-hangzhou.aliyuncs.com/chenby/#g" *.yaml;
sed -i "s#k8s.gcr.io/prometheus-adapter/#registry.cn-hangzhou.aliyuncs.com/chenby/#g" *.yaml;
sed -i "s#quay.io/prometheus-operator/#registry.cn-hangzhou.aliyuncs.com/chenby/#g" *.yaml;
sed -i "s#k8s.gcr.io/kube-state-metrics/#registry.cn-hangzhou.aliyuncs.com/chenby/#g" *.yaml;
sed -i "/ports:/i\  type: NodePort" grafana-service.yaml;
sed -i "/targetPort: http/i\    nodePort: 31100" grafana-service.yaml;
sed -i "/ports:/i\  type: NodePort" prometheus-service.yaml;
sed -i "/targetPort: web/i\    nodePort: 31200" prometheus-service.yaml;
sed -i "/targetPort: reloader-web/i\    nodePort: 31300" prometheus-service.yaml;
sed -i "/ports:/i\  type: NodePort" alertmanager-service.yaml;
sed -i "/targetPort: web/i\    nodePort: 31400" alertmanager-service.yaml;
sed -i "/targetPort: reloader-web/i\    nodePort: 31500" alertmanager-service.yaml;
kubectl create -f /root/kube-prometheus/manifests/setup;
kubectl create -f /root/kube-prometheus/manifests/;
sleep 30;
kubectl get pod -n monitoring;
kubectl get svc -n monitoring;
```



Alertmanager: http://192.168.3.125:31400
Prometheus: http://192.168.3.125:31200/
Grafana: http://192.168.3.125:31100/