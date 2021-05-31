```
# --generator=run-pod/v1 参数即将被废弃
# kubectl 现在应该不支持创建 deployment
kubectl run --generator=run-pod/v1 --image=gcr.io/kuar-demo/kuard-amd64:1 kuard

# 可以通过 --dry-run -o yaml 确定 kubectl run 创建的 manifest 
kubectl run --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --dry-run -o yaml

# 支持 pod, deployment, service 
kubectl port-forward kuard 8080:8080

# Running a service
kubectl expose deployment kuard --type=LoadBalancer --port=80 --target-port=8080 --dry-run -o yaml
```

