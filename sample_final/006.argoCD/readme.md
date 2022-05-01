B1. Cài đặt
```
kubectl create namespace argocd
curl -o https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml argocd.yaml
kubectl apply -n argocd -f argocd.yaml
kubectl apply -n argocd -f argocd-nodeport.yaml
```

B2: lấy mật khẩu đăng nhập:
```
kubectl -n argocd get secret argocd-initial-admin-secret 
```