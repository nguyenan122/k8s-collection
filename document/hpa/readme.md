#Install kube-metric
https://github.com/nguyenan122/k8s-collection/tree/main/sample_final/kube-metric

# Create Deployment:
> k create deployment nginx-hpa --image=nginx:alpine --replicas=2 --port=80 --dry-run=client -o yaml > deployment.yaml
edit yaml file add limit and request CPU
```
       resources: 
          limits:
            cpu: 50m
          requests:
            cpu: 20m
```

# Create Services
> k expose deployment nginx-hpa --name=nginx-svc --port=80 --target-port=80 --type=NodePort --dry-run=client -o yaml > svc.yaml


# Create HPA
> k autoscale deployment nginx-hpa --min=2 --max=10 --cpu-percent=10 --dry-run=client -o yaml > hpa.yaml



Stress test:
```
sudo yum install httpd-tools
ab -k -c 200 -n 2000000 http://localhost:$(k get svc | grep nginx | awk '{print $5}' | awk -F ':' '{print $2}' | awk -F '/' '{print $1}')/
```

