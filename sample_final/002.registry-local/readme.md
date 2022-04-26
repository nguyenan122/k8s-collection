**Bước 1: Chỉ định hosts trên các worker_node và master_node:**  
```
echo 192.168.88.12 registry.tuanda.vn >> /etc/hosts
```
**Bước 2: Import basic-auth và ssl vào configmap** 
```
mkdir /opt/certs /opt/registry
cd /opt/certs
openssl req -x509 -out ca.crt -keyout ca.key -days 1825 \
  -newkey rsa:2048 -nodes -sha256 \
  -subj '/CN=registry.tuanda.vn' -extensions EXT -config <( \
   printf "[dn]\nCN=registry.tuanda.vn\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:registry.tuanda.vn\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")

cd /opt/certs/
kubectl create namespace registry
kubectl -n registry create configmap registry-cert --from-file=ca.crt --from-file=ca.key
sudo yum install httpd-tools -y ; htpasswd -Bbn tuanda 123 > htpasswd
kubectl -n registry create configmap registry-basic-auth --from-file=htpasswd
kubectl -n registry get configmaps 
```

**Bước 3: Tạo deployment và service NodePort** 
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
      storage: 10Gi

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
        configMap:
          name: registry-cert
      - name: auth-vol
        configMap:
          name: registry-basic-auth
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
            value: "/certs/ca.crt"
          - name: REGISTRY_HTTP_TLS_KEY
            value: "/certs/ca.key"
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
  
  
# kubectl -n registry apply -f deployment.yaml 
```
**Bước 4: Add Trust CA Self certificate trên tất cả các node (all node)** 
```
sudo cp -rp /opt/certs/ca.crt  /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
sudo service docker restart
```
**Bước 5: Đẩy cert vào tất cả các node docker, để permit self-certificate gọi pull. (all node)** 
```
mkdir -p /etc/docker/certs.d/registry.tuanda.vn:31320
cp -rp /opt/certs/ca.crt /etc/docker/certs.d/registry.tuanda.vn\:31320/
```
**Bước 6: docker login đẩy config registry client sang các node: (all node)** 
```
# curl -v --user tuanda:123 https://registry.tuanda.vn:31320/v2/
# docker login registry.tuanda.vn:31320 -u tuanda -p 123
cat ~/.docker/config.json 
{
	"auths": {
		"registry.tuanda.vn:31320": {
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
# docker tag nginx:alpine registry.tuanda.vn:31320/nginx:alpine
# docker push registry.tuanda.vn:31320/nginx:alpine
```

**Bước 8: Launch pod với option registry**
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
      containers:
      - name: hello-kubernetes-debug
        image: admin.tuan.name.vn:31320/debug-tools:1.0.0
        ports:
        - containerPort: 8080
      - name: hello-kubernetes-nginx
        image: admin.tuan.name.vn:31320/nginx:alpine
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: regcred
```
