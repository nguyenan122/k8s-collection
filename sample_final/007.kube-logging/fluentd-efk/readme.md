
```
kubectl appply -f 01.ns.yaml
kubectl appply -f 02.svc.yaml
kubectl appply -f 03.sts_elasticsearch.yaml
# check ES by curl: curl -s http://localhost:30092/_cluster/state?pretty | grep es-cluster
kubectl appply -f 04.kibana.yaml
kubectl appply -f 05.fluentd.yaml
kubectl appply -f 06.pod-test-log.yaml
# Access kibana by browser: http://${YOUR_IP_K8S_CLUSTER}:30100/


```


Tham Khảo:

[EFK DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes)
[EFK Medium](https://azmifarih.medium.com/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes-1c455a6f17a8)