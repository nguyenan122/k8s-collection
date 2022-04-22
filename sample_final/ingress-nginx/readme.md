Nginx có 2 loại ingress phổ biến do k8s và nginx viết:
- https://kubernetes.github.io/ingress-nginx/deploy/
- https://docs.nginx.com/nginx-ingress-controller/ 

```
curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml
kubectl apply -f deploy.yaml
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo
kubectl create ingress demo-localhost --class=nginx --rule=demo.localdev.me/*=demo:80
--- Hoặc dùng Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install ingress-nginx bitnami/nginx-ingress-controller --namespace=ingress-nginx
```
Kiểm tra ingress đã được tạo
```
[tuanda@master-node ~]$ k get ingress -A
NAMESPACE   NAME             CLASS   HOSTS              ADDRESS   PORTS   AGE
default     demo-localhost   nginx   demo.localdev.me             80      10m

[tuanda@master-node ~]$ k get ingress demo-localhost -o yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-localhost
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: demo.localdev.me
    http:
      paths:
      - backend:
          service:
            name: demo
            port:
              number: 80
        path: /
        pathType: Prefix
status:
  loadBalancer: {}
```