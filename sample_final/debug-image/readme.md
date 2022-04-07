Image_1:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: debugpod
  name: debugpod
spec:
  containers:
  - image: nicolaka/netshoot
    name: debugpod
    command: ["/bin/bash"]
    args: ["-c", "sleep 100000"]
```

Image_2:
```
Dockerfile.app
FROM alpine:latestRUN apk update && \
    apk --no-cache add \
         bash \
         curl
Dockerfile.debug
FROM alpine:latestRUN apk update && \
    apk --no-cache add \
         bash \
         tcpdumpCMD exec /bin/bash -c "trap : TERM INT; sleep infinity & wait"
and build them:
> docker build --no-cache --progress=plain -f docker\Dockerfile.app -t do-wget:1.0.0 docker\
> docker build --no-cache --progress=plain -f docker\Dockerfile.debug -t debug-tools:1.0.0 docker\
```


Nguồn: 
https://cloudogu.com/en/blog/k8s-app-ops-part-2 , https://github.com/nicolaka/netshoot
https://pauldally.medium.com/debugging-networkpolicy-part-3-83658d26747e 