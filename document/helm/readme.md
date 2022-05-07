# Cài đặt:
```
https://helm.sh/docs/intro/install/ 
curl -k -O https://get.helm.sh/helm-canary-linux-amd64.tar.gz
tar -xvzf helm-canary-linux-amd64.tar.gz
chmod 7550 linux-amd64/helm ; mv linux-amd64/helm /usr/bin/
```

#Bash Complete for helm
```
yum install bash-completion 
helm completion bash > /etc/bash_completion.d/helm
```


# Phần 1: Làm quen
```
# Add Repo
helm repo add bitnami https://charts.bitnami.com/bitnami --force-update
helm repo add stable https://charts.helm.sh/stable --force-update
helm repo update
helm search repo nginx
helm repo list
```

Tải helm .tar.gz từ ArtifactHub
```
helm pull bitnami/kube-state-metrics
```

Chạy helm
```
helm install kube-state-metrics bitnami/kube-state-metrics -n tuanda
helm ls -A
helm status first-chart
helm delete kube-state-metrics -n tuanda
```

Show thông tin của chart trên ArtifactHub
```
helm show chart bitnami/nginx
helm show values bitnami/nginx
```

Liệt kê helm đang chạy (-n là namespace)
```
helm ls -n tuanda
helm ls -A
```
Đặt thêm biến - tự chỉ định khi chạy helm
```
helm install nginxtest bitnami/nginx --set service.port=8888 -n tuanda
```



# Phần 2: Tự tạo Helm
Tạo self chart
```
helm create first-chart
```
Xóa tạm thư mục templates đi. Tạo file configmap.yaml để test
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config
  namespace: default
data:
  sleep-interval: "25"
```
Để chạy helm ta vừa sửa
```
helm install first-chart .
kubectl get configmaps fortune-config
kubectl describe configmaps fortune-config
```


Nếu Chart có thêm update ta dùng upgrade để cập nhập:
```
helm template first-chart . --namespace=tuanda
helm upgrade first-chart .
kubectl describe configmaps fortune-config
```

Để Rolback ta làm như sau
```
helm history ${helm_name}
helm rollback ${helm_name} 1
```

Nếu thêm secret trong chart như configmap
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  USER_NAME: YWRtaW4=
  PASSWORD: MWYyZDFlMmU2N2Rm
helm upgrade first-chart .
```


# Phần 3: Link variable vào Chart.yaml và variable.yaml

3.1 Chart.yaml

Sửa file configmap ở phần 1:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config-{{.Chart.Version}}
  namespace: default
data:
  sleep-interval: "25"
  tuanda: "ahihi2"
helm upgrade first-chart .
[tuanda@master-node first-chart]$ kubectl get configmaps 
NAME                   DATA   AGE
fortune-config-0.1.0   2      68s
```
3.2 Values.yaml
```
Sửa file:
tuanda:
  ahihi: "xin chao"

replicaCount: 1
Ta có {{.Values.tuanda.ahihi}}
```
3.3 Dynamic if/else in helm
```
tuanda@master-node first-chart]$ cat templates/secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  {{if eq .Values.env "Staging"}}
  USER_NAME: Staging
  PASSWORD: M12345G
  {{else}}
  USER_NAME: YWRtaW4=
  PASSWORD: MWYyZDFlMmU2N2Rm
  {{end}}
```
Thêm env: "Staging" trong file values.yaml và test
```
helm template first-chart .
helm upgrade first-chart .
```

# Phần 4: Helm push local registry
```
# export HELM_EXPERIMENTAL_OCI=1
# helm registry login -u tuanda registry.tuan.name.vn:31320
# helm registry logout registry.tuan.name.vn:31320

Đẩy .tar.gz lên local registry, sau đó test pull lại (Để ý kĩ sẽ thấy md5sum push/pull không đổi):
# helm push nginx-test.tar.gz oci://registry.tuan.name.vn:31320/helm-charts
# helm pull oci://registry.tuan.name.vn:31320/helm-charts/nginx-test
```


