**Mô hình:**

[NFS-server]---[Master]----[Worker]----[Worker]

**B1.1: Cài Nfs server:**
> Ta có thể cài theo ansible đã có sẵn cũng đc. Chỉ cần sửa lại inventory_host.yml là xong. Thay cho cả 2 bước 1.1 và 1.2
> ansible-playbook -i inventory_nfs.yml nfs.yml

```
yum install nfs-utils nfs-utils-lib -y
chkconfig rpcbind on
chkconfig nfs on 
service rpcbind restart
service nfs restart
mkdir -p /data/nfs-k8s/ ; vim /etc/exports
/data/nfs-k8s/ 192.168.88.0/24(rw,sync,subtree_check,no_root_squash)
exportfs -a
showmount -e 192.168.88.88
```

**B1.2: Cài nfs client trên tất cả worker node, nếu không sẽ lỗi không mout đc vào pod.**
```
yum install nfs-utils nfs-utils-lib -y
chkconfig nfs off
chkconfig rpcbind off
```


**B2: Helm install**
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm pull nfs-subdir-external-provisioner/nfs-subdir-external-provisioner
kubectl create ns nfs-provisioner
helm install nfs-provisioner-retain . --set nfs.server=192.168.88.12 \
  --set nfs.path=/data/retain \
  --set storageClass.name=nfs-provisioner-retain \
  --set storageClass.onDelete=retain \
  --set storageClass.accessModes=ReadWriteMany \
  --namespace=nfs-provisioner
  
helm install nfs-provisioner-delete . --set nfs.server=192.168.88.12 \
  --set nfs.path=/data/delete \
  --set storageClass.name=nfs-provisioner-delete \
  --set storageClass.onDelete=delete \
  --set storageClass.accessModes=ReadWriteMany \
  --namespace=nfs-provisioner
```

**B4: Tạo PVC và Pod theo Storage đã thêm**
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sc-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-provisioner-retain
  resources:
    requests:
      storage: 2Gi
```

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  volumes:
  - name: myvol
    persistentVolumeClaim:
      claimName: sc-nfs-pvc
  containers:
  - image: busybox
    name: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 600000"]
    volumeMounts:
    - name: myvol
      mountPath: /data
```

Nguồn:
https://fabianlee.org/2022/01/12/kubernetes-nfs-mount-using-dynamic-volume-and-storage-class/?msclkid=a5ca54e2ae7111eca259ced5d2223a6f
https://artifacthub.io/packages/helm/nfs-subdir-external-provisioner/nfs-subdir-external-provisioner 
