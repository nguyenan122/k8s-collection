Nguồn: [k8s Taint and Toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

# nodeName
khai báo trực tiếp worker-node được chọn
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx-schedule
  name: nginx-schedule
spec:
  containers:
  - image: nginx:alpine
    name: nginx-schedule
    resources: {}
  nodeName: worker-node2
```

# nodeSelector
Dựa theo nhãn labels của worker-node
```
# k label nodes worker-node1 vga=enable_vga --overwrite
# k label nodes worker-node2 vga=disable_vga --overwrite

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx-schedule
  name: nginx-schedule
spec:
  containers:
  - image: nginx:alpine
    name: nginx-schedule
    resources: {}
  nodeSelector:
    vga: enable_vga

```


# Taints and tolerations
Mục đích: Chặn pod đc deploy. Nhằm tách môi trường (test/production), (vga/no-vga) hoặc (frontend/backend) mà pod đc deploy tới.
Taint gán cho worker-node (không phải cho pod). Có 2 loại taint:
- NoSchedule: pod mới phải thỏa mãn yêu cầu mới đc deploy, pod cũ KO bị ảnh hưởng
- PreferNoSchedule: nếu pod không thể deploy lên worker nào, thì cuối cùng pod có thể deploy lên nó.
- NoExecute: pod mới và pod cũ PHẢI thỏa mãn yêu cầu mới đc deploy. Pod cũ sẽ bị xóa.
```
$ kubectl describe node master-node  | grep -i taint
Taints:             node-role.kubernetes.io/master:NoSchedule
$ kubectl describe node worker-node1  | grep -i taint
Taints:             <none>

```
Gán taint và untaint
```
kubectl taint nodes worker-node1 node-type-abc=production:NoSchedule
kubectl taint nodes worker-node1 node-type-abc=production:NoSchedule-
```
Gán Toleration cho Pod
```
# kubectl taint nodes worker-node1 node-type-abc=production:NoSchedule

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx-schedule
  name: nginx-schedule
spec:
  containers:
  - image: nginx:alpine
    name: nginx-schedule
    resources: {}
  tolerations:
  - key: "node-type-abc"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"

```
- Chức năng auto của Taint trong k8s, ví dụ như node bị đánh là down, thì node đó tự động được gán Taint = node.kubernetes.io/not-ready:NoExecute (trên trang có rất nhiều loại). Ta có thể dựa vào đây để ép pod được deploy sang node khác nếu node hiện tại bị lỗi.

# Node affinity and Pod affinity
## A. Node affifity
Mục đích: Ưu tiên pod được deploy vào node nào. (ngoài ra còn có thể chia tải weight số pod được deploy trên từng node).
- requiredDuringSchedulingIgnoredDuringExecution: áp dụng KHI schedule, KHÔNG áp dúng  pod cũ đã deploy.
- requiredDuringSchedulingRequiredDuringExecution: áp dụng KHI schedule, CÓ áp dúng  pod cũ đã deploy.
- preferredDuringSchedulingIgnoredDuringExecution: ưu tiên KHI schedule, KHÔNG áp dúng  pod cũ đã deploy.
- preferredDuringSchedulingRequiredDuringExecution: ưu tiên KHI schedule, CÓ áp dúng  pod cũ đã deploy.

Nguồn [k8s affinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)
```
# kubectl label nodes worker-node1 disktype=ssd
# kubectl label nodes worker-node2 disktype=hdd
# k get node --show-labels
Tạo pod yêu cầu worker PHẢI có ổ cứng là SSD, ưu tiên deploy ở zone1-2
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
			- nvme
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: Zone
            operator: In
            values:
            - Zone 1
            - Zone 2			
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```
VD2: về kết hợp require và prefer. Deploy pod trên node có ổ cứng là ssd, với ssd nvme_class3 thì triển khai 50% pod....
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: webconsole
spec:
  replicas: 10
  selector:
    matchLabels:
      app: webconsole
  template:
    metadata:
      labels:
        app: webconsole
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              preference:
                matchExpressions:
                - key: disktype
                  operator: In
                  values:
                    - "nvme_class3"
            - weight: 30
              preference:
                matchExpressions:
                - key: disktype
                  operator: In
                  values:
                    - "nvme_class1"
            - weight: 20
              preference:
                matchExpressions:
                - key: disktype
                  operator: In
                  values:
                    - "nvme_class2"
      containers:
        - image: nginx:alpine
          name: nginx
```

## B. Pod affinity và Pod anti-affifity
B1. Pod affinity
Mục đích: deploy backend gần db, frontend xa backend...(pod vs pod)
NGuồn: [k8s pod affifity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: kubernetes.io/hostname
              labelSelector:
                  matchLabels:
                    app: database
      containers:
        - name: main
          image: alpine
          command: ["/bin/sleep", "999999"]
```

B2. Pod anti-affifity
Mục đích: chặn pod frontend chuẩn bị deploy tránh xa backend
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: kubernetes.io/hostname
              labelSelector:
                  matchLabels:
                    app: backend
      containers:
        - name: main
          image: alpine
          command: ["/bin/sleep", "999999"]
```


----------------------
Nguồn: 
- [k8s Taint and Toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Viblo Quân Huỳnh](https://viblo.asia/p/kubernetes-series-bai-17-advanced-scheduling-taints-and-tolerations-924lJBDalPM)
