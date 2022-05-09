# Configmap
Config8 map được tạo từ các loại sau:
```
--from-literal=sleep-interval=25
--from-file=customkey=my-nginx-config.conf
--from-env-file=path/to/foo.env
```
Ta có thể mount config map vào ENV hoặc Volume

VD1: Mout vào ENV:
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: nginx:alpine
    env:
    - name: INTERVAL
      valueFrom: 
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
```

VD2: Mount vào Volume:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  index.html: |
    Xinchao!!!
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  selector:
    matchLabels:
      app: test
  replicas: 1
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: configmaptestapp
        image: nginx:alpine
        volumeMounts:
        - mountPath: /usr/share/nginx/html/index.html
          name: data-volume
        ports:
        - containerPort: 80
      volumes:
        - name: data-volume
          configMap:
            name: test-config

```

# Secret
Secret có 3 loại
```
docker-registry: rành riêng cho docker registry.
generic: secret thường
tls: liên quan đến certificate            
```
[https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/)