B1. Tải file về apply
```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

B2. Fix lỗi
(Nguồn: https://www.linuxsysadmins.com/service-unavailable-kubernetes-metrics/ )
```
# kubectl top pods
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get pods.metrics.k8s.io)
```
Với Master-node ta chỉ cần sửa như sau
```
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-insecure-tls=true          #THÊM CÁI NÀY
        - --kubelet-preferred-address-types=InternalIP
```
Với worker-node ta cần sửa thêm:
```
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      hostNetwork: true                        #THÊM CÁI NÀY
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
```
