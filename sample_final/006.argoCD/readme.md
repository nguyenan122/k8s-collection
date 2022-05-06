B1. Cài đặt
```
kubectl create namespace argocd
curl -o https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml argocd.yaml
kubectl apply -n argocd -f argocd.yaml
kubectl apply -n argocd -f argocd-nodeport.yaml

-----Hoặc cài từ helm-----
helm repo add argo https://argoproj.github.io/argo-helm
helm install --name argo-cd argo/argo-cd \
  --set server.service.type=NodePort

```

B2: lấy mật khẩu đăng nhập user admin:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

B3: Cài đặt Argocd-cli
```
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
sudo curl --silent --location -o /usr/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
sudo chmod +x /usr/bin/argocd

#Argocd bash completion
echo "source <(argocd completion bash)" >> ~/.bash_profile
```

B4: Login Argocd
```
argocd login 192.168.88.12:31321 --username admin --password $ARGO_PWD --insecure --grpc-web
```

B5: Create App
```
argocd app create nginx --repo https://github.com/NiharikaNallabelli/argocd.git --path dev --dest-namespace default --dest-server https://kubernetes.default.svc
```


B5: Other command
```
argocd app list


```



Nguồn tham khảo:

[Medium Argocd](https://medium.com/zelarsoft/using-gitops-with-argocd-b75c54ab4738)

[Dev.to]()