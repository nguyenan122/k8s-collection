
# Create Deployment:
> k create deployment nginx-hpa --image=nginx:alpine --replicas=2 --port=8888 --dry-run=client -o yaml > deployment.yaml

# Create Services
> k create service nodeport svc-tomcat-hpa --tcp=8888:8888 --node-port=31101 --dry-run=client -o yaml
edit yaml file add limit and request CPU
```
       resources: 
          limits:
            cpu: 50m
          requests:
            cpu: 20m
```

# Create HPA
> k autoscale deployment tomcat-hpa --max=10 --min=2 --cpu-percent=5 --dry-run=client -o yaml > hpa.yaml



Stress test:

