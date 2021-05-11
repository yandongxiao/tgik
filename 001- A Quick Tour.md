```
# This tells kubecfg to read its config from the local directory
export KUBECONFIG=./kubeconfig

# Looking at the cluster
kubectl get nodes
kubectl get pods --namespace=kube-system

# Running a single pod
kubectl run --generator=run-pod/v1 --image=gcr.io/kuar-demo/kuard-amd64:1 kuard
kubectl get pods
kubectl run --generator=run-pod/v1 --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --dry-run -o yaml
kubectl get pods kuard -o yaml
kubectl port-forward kuard 8080:8080
kubectl delete pod kuard

# Running a deployment
kubectl run --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --replicas=5 --dry-run -o yaml
kubectl run --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --replicas=5
kubectl get pods

# Running a service
kubectl expose deployment kuard --type=LoadBalancer --port=80 --target-port=8080 --dry-run -o yaml
kubectl get service kuard -o wide

# Wait a while for the ELB to be ready
KUARD_LB=$(kubectl get service kuard -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')

---
# Doing a deployment
## Window 1
## On the mac, install with `brew install watch`
watch -n 0.1 kubectl get pods

## Window 2
## On the mac, install with `brew install jq`
while true ; do curl -s http://${KUARD_LB}/env/api | jq '.env.HOSTNAME'; done

## Window 3
kubectl scale deployment kuard --replicas=10
kubectl set image deployment kuard kuard=gcr.io/kuar-demo/kuard-amd64:2
kubectl rollout undo deployment kuard

---
# Clean up
kubectl delete deployment,service kuard
```

