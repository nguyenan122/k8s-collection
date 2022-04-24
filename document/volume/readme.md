Volume có nhiều loại, chủ yếu ta dùng EmptyDir và Hostpath (https://kubernetes.io/docs/concepts/storage/volumes/ )

# 1. EmptyDir
Ví dụ về share emptydir để dùng làm trung gian giữa 2 image:
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```


# HostPath
HostPath sẽ lưu data ngay tại worker node, nên dữ liệu chỉ được lưu trên worker-node đó. Để khắc phục ta có thể share mount NFS thư mục - các node.
```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    hostPath:
      path: /tmp/mongodb
```
