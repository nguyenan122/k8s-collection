






```
# k apply -f application.yaml 

# k -n myapp get all
NAME                       READY   STATUS    RESTARTS   AGE
pod/nginx-65778599-6z9wc   1/1     Running   0          9m2s
pod/nginx-65778599-hsbkq   1/1     Running   0          9m2s
pod/nginx-65778599-qvjnl   1/1     Running   0          9m2s

NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/nginx   NodePort   10.103.134.165   <none>        80:31323/TCP   3m48s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   3/3     3            3           9m2s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-65778599   3         3         3       9m2s

# argocd app list
```
