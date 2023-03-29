Nginx có 2 loại ingress phổ biến do k8s và nginx viết:
- https://kubernetes.github.io/ingress-nginx/deploy/
- https://docs.nginx.com/nginx-ingress-controller/ 


Cách 1: Cài bằng helm

```
B1. Cài đặt helm3 nếu chưa có
wget https://get.helm.sh/helm-v3.10.3-linux-amd64.tar.gz
tar -zxvf helm-v3.10.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/
rm -rf helm-v3.10.3-linux-amd64.tar.gz linux-amd64

B2. Tải helm chart về
helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/nginx-ingress-controller
tar -xvzf nginx-ingress-controller-9.4.1.tgz
cd nginx-ingress-controller/

B3: Sửa lại ServiceType và Port
vim values.yaml
service:
  type: NodePort
  nodePorts:
    http: "30080"
    https: "30443"


B4: Chạy helm chart
helm -n ingress install ingress -f values.yaml . --create-namespace
```

Cách 2: Cài trực tiếp từ git k8s
```
# curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
# kubectl apply -f deploy.yaml
# kubectl create deployment demo --image=httpd --port=80
# kubectl expose deployment demo
# kubectl create ingress demo-localhost --class=nginx --rule=demo.localdev.me/*=demo:80

--- Hoặc dùng Helm
# helm repo add bitnami https://charts.bitnami.com/bitnami
# helm pull bitnami/nginx-ingress-controller
# cat values.yaml  | grep replicaCount #(có thể sửa số replica trước khi install helm)
# helm install ingress-nginx . --namespace=ingress-nginx --create-namespace
# trường hợp có thay đổi value của helm, ta có thể upgrade: helm upgrade ingress-nginx . --namespace=ingress-nginx

```
Kiểm tra ingress đã được tạo
```
# kubectl create deployment demo --image=httpd --port=80
# kubectl expose deployment demo
# kubectl create ingress demo-localhost --class=nginx --rule=demo.localdev.me/*=demo:80

# curl -H 'Host: demo.localdev.me' localhost:31240
<html><body><h1>It works!</h1></body></html>

```