curl -O https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml
Ta sửa lại Services về NodePort
kubectl apply -f recommended.yaml
truy cập vào https://IP:Nodeport
Tạo Token: 
```
kubectl create serviceaccount dashboard-admin-sa
kubectl create clusterrolebinding dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
kubectl get secrets | grep dashboard-admin-sa-token
k get secrets dashboard-admin-sa-toke.... –o yaml | grep token
Ta lấy token từ secret, sau đó base64 decode là có token.
```