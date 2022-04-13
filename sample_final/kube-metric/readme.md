B1. Tải file về apply
```
curl https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -o kube-metric.yaml
```

B2. Fix lỗi
(Nguồn: https://www.linuxsysadmins.com/service-unavailable-kubernetes-metrics/ )
```
# kubectl top pods
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get pods.metrics.k8s.io)
```
Ta sửa bằng
```
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-insecure-tls=true          #THÊM CÁI NÀY
        - --kubelet-preferred-address-types=InternalIP
```

