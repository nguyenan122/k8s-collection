Ta sẽ chia làm 2 phần. Phần 1 là có certificate xịn, phần 2 là chỉ có self certitifcate.

# PHẦN 1: Tạo registry có cert xịn

**Bước 1: Chỉ định hosts trên các worker_node và master_node, nếu có DNS trỏ thì không cần bước này**  
```
sudo echo 192.168.88.12 registry.tuan.name.vn >> /etc/hosts
```
**Bước 2: Import SSL / acc vào hệ thống**
```
mkdir /opt/certs /opt/registry
Đẩy cert ssl của domain registry.tuan.name.vn vào thư mục /opt/cert
# cd /opt/certs/
# kubectl create namespace registry
# kubectl -n registry create secret tls registry-cert --key=ca.key --cert=ca.crt
# sudo yum install httpd-tools -y
# htpasswd -Bbn tuanda 123 > htpasswd
# kubectl -n registry create secret generic registry-basic-auth --from-file=htpasswd
# kubectl -n registry get secrets 
```

**Bước 3: Tạo deployment và service NodePort**

Chú ý: trước bước này, bạn phải tạo nfs-provision để mount vào PVC Registry (https://github.com/nguyenan122/k8s-collection/tree/main/sample_final/001.nfs-provisioning)
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: registry   
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-provisioner-retain
  resources:
    requests:
      storage: 3Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-repository-k8s
  labels:
    app: private-repository-k8s
  namespace: registry    
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-repository-k8s
  template:
    metadata:
      labels:
        app: private-repository-k8s
    spec:
      containers:
        - image: registry:2
          name: private-repository-k8s
          imagePullPolicy: IfNotPresent
          env:
          - name: REGISTRY_AUTH
            value: htpasswd
          - name: REGISTRY_AUTH_HTPASSWD_PATH
            value: "/auth/htpasswd"
          - name: REGISTRY_AUTH_HTPASSWD_REALM
            value: Registry Realm
          - name: REGISTRY_HTTP_TLS_CERTIFICATE
            value: "/certs/tls.crt"
          - name: REGISTRY_HTTP_TLS_KEY
            value: "/certs/tls.key"
          ports:
            - containerPort: 5000
          volumeMounts:
          - name: certs-vol
            mountPath: /certs
          - name: registry-vol
            mountPath: /var/lib/registry
          - name: auth-vol
            mountPath: /auth
      volumes:
      - name: certs-vol
        secret:
          secretName: registry-cert
      - name: auth-vol
        secret:
          secretName: registry-basic-auth
      - name: registry-vol
        persistentVolumeClaim:
          claimName: registry-pvc            
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: private-repository-k8s
  name: private-repository-k8s
  namespace: registry  
spec:
  ports:
  - port: 5000
    nodePort: 31320
    protocol: TCP
    targetPort: 5000
  selector:
    app: private-repository-k8s
  type: NodePort
  
Tạo 3 file với các nội dung trên. sau đó ta apply:  
# kubectl apply -f pvc.yaml
# kubectl apply -f deployment.yaml
# kubectl apply -f svc.yaml

Kiểm tra gọi registry:
# curl -u 'tuanda:123' https://registry.tuan.name.vn:31320/v2/_catalog
> Kết quả json: {"repositories":[]}
```

**Bước 4: Docker login trên máy thông ra ngoài docker-hub internet, để kéo image về, đẩy lên local-registry (là 1 máy trung gian cũng đc, không cần phải là cụm k8s)** 

Mô hình: Docker-hub --> Máy Trung gian --> Docker local
```
# curl -u 'tuanda:123' https://registry.tuan.name.vn:31320/v2/_catalog
# docker login registry.tuan.name.vn:31320 -u tuanda -p 123
cat ~/.docker/config.json 
{
	"auths": {
		"registry.tuan.name.vn:31320": {
			"auth": "dHVhbmRhOjEyMw=="
		}
	}
}
mkdir -p /home/tuanda/.docker ;  chown -R tuanda.tuanda /home/tuanda/.docker
Ta copy file config.json ở trên sang các worker node trong cluster. (/home/tuanda/.docker/config.json)
```
Đẩy image kéo về lên local registry
```
# docker pull nginx:alpine
# docker tag nginx:alpine registry.tuan.name.vn:31320/nginx:alpine
# docker push registry.tuan.name.vn:31320/nginx:alpine
```

**Bước 8: Launch pod với option registry**
```
# kubectl create ns tuanda

# kubectl -n tuandacreate secret docker-registry private-reg-cred --docker-username=tuanda --docker-password=123 --docker-server=registry.tuan.name.vn:31320 --docker-email=ahihi@registry.tuan.name.vn
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
  namespace: tuanda
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      imagePullSecrets:
      - name: private-reg-cred
      containers:
      - name: hello-kubernetes-nginx
        image: registry.tuan.name.vn:31320/nginx:alpine
        ports:
        - containerPort: 80
```






# PHẦN 2 Registry với Self-certificate

**Bước 1: Chỉ định hosts trên các worker_node và master_node, nếu có DNS trỏ thì không cần bước này**  
```
sudo echo 192.168.88.12 registry.tuan.name.vn >> /etc/hosts
```
**Bước 2.1: Tạo ssl key, trường hợp có rồi, ta có thể bỏ qua bước này** 
```
mkdir /opt/certs /opt/registry
chown -R tuanda.tuanda /opt/certs /opt/registry
su - tuanda ; cd /opt/certs
openssl req -x509 -out ca.crt -keyout ca.key -days 1825 \
  -newkey rsa:2048 -nodes -sha256 \
  -subj '/CN=registry.tuan.name.vn' -extensions EXT -config <( \
   printf "[dn]\nCN=registry.tuan.name.vn\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:registry.tuan.name.vn\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```
**Bước 2.2: Import SSL / acc vào hệ thống**
```
mkdir /opt/certs /opt/registry
cd /opt/certs/
kubectl create namespace registry
kubectl -n registry create secret tls registry-cert --key=ca.key --cert=ca.crt
sudo yum install httpd-tools -y
htpasswd -Bbn tuanda 123 > htpasswd
kubectl -n registry create secret generic registry-basic-auth --from-file=htpasswd
kubectl -n registry get secrets 
```

**Bước 3: Tạo deployment và service NodePort**

Chú ý: trước bước này, bạn phải tạo nfs-provision để mount vào PVC Registry (https://github.com/nguyenan122/k8s-collection/tree/main/sample_final/001.nfs-provisioning)
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: registry   
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-provisioner-retain
  resources:
    requests:
      storage: 3Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-repository-k8s
  labels:
    app: private-repository-k8s
  namespace: registry    
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-repository-k8s
  template:
    metadata:
      labels:
        app: private-repository-k8s
    spec:
      volumes:
      - name: certs-vol
        secret:
          secretName: registry-cert
      - name: auth-vol
        secret:
          secretName: registry-basic-auth
      - name: registry-vol
        persistentVolumeClaim:
          claimName: registry-pvc
      containers:
        - image: registry:2
          name: private-repository-k8s
          imagePullPolicy: IfNotPresent
          env:
          - name: REGISTRY_AUTH
            value: htpasswd
          - name: REGISTRY_AUTH_HTPASSWD_PATH
            value: "/auth/htpasswd"
          - name: REGISTRY_AUTH_HTPASSWD_REALM
            value: Registry Realm
          - name: REGISTRY_HTTP_TLS_CERTIFICATE
            value: "/certs/tls.crt"
          - name: REGISTRY_HTTP_TLS_KEY
            value: "/certs/tls.key"
          ports:
            - containerPort: 5000
          volumeMounts:
          - name: certs-vol
            mountPath: /certs
          - name: registry-vol
            mountPath: /var/lib/registry
          - name: auth-vol
            mountPath: /auth
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: private-repository-k8s
  name: private-repository-k8s
  namespace: registry  
spec:
  ports:
  - port: 5000
    nodePort: 31320
    protocol: TCP
    targetPort: 5000
  selector:
    app: private-repository-k8s
  type: NodePort
  
  
# kubectl apply -f pvc.yaml
# kubectl apply -f deployment.yaml
# kubectl apply -f svc.yaml

Kiểm tra gọi registry:
# curl -u 'tuanda:123' https://registry.tuan.name.vn:31320/v2/_catalog
> Kết quả json: {"repositories":[]}
```
**Bước 4: Add Trust CA Self certificate trên tất cả các node (all node) -nếu là self-cert, còn cert xịn thì ko cần** 
```
sudo cp -rp /opt/certs/ca.crt  /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
sudo service docker restart

scp -r /opt/certs root@${WORKER_IP}:/opt/
ssh root@${WORKER_IP}
sudo cp -rp /opt/certs/ca.crt  /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
cat /etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt
sudo service docker restart
```
**Bước 5: Đẩy cert vào tất cả các node docker, để permit self-certificate gọi pull. (all node)-nếu là self-cert, còn cert xịn thì ko cần** 
```
sudo mkdir -p /etc/docker/certs.d/registry.tuanda.vn:31320
sudo cp -rp /opt/certs/ca.crt /etc/docker/certs.d/registry.tuanda.vn\:31320/
```
**Bước 6: docker login đẩy config registry client sang các worker node: (worker node)** 
```
# curl -u 'tuanda:123' https://registry.tuan.name.vn:31320/v2/_catalog
# docker login registry.tuan.name.vn:31320 -u tuanda -p 123
cat ~/.docker/config.json 
{
	"auths": {
		"registry.tuan.name.vn:31320": {
			"auth": "dHVhbmRhOjEyMw=="
		}
	}
}
mkdir -p /home/tuanda/.docker ;  chown -R tuanda.tuanda /home/tuanda/.docker
Ta copy file config.json ở trên sang các worker node trong cluster. (/home/tuanda/.docker/config.json)
```
**Bước 7: đẩy image lên registry:** 
```
# docker pull nginx:alpine
# docker tag nginx:alpine registry.tuan.name.vn:31320/nginx:alpine
# docker push registry.tuan.name.vn:31320/nginx:alpine
```

**Bước 8: Launch pod với option registry**
```
kubectl create ns tuanda

kubectl -n tuandacreate secret docker-registry private-reg-cred --docker-username=tuanda --docker-password=123 --docker-server=registry.tuan.name.vn:31320 --docker-email=ahihi@registry.tuan.name.vn
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
  namespace: tuanda
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      imagePullSecrets:
      - name: private-reg-cred
      containers:
      - name: hello-kubernetes-nginx
        image: registry.tuan.name.vn:31320/nginx:alpine
        ports:
        - containerPort: 80
```
