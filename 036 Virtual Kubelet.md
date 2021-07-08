# 036: Virtual Kubelet

## 什么是 virtual kubelet？

来自 virtual-kubelet 命令行的解释。virtual-kubelet implements the Kubelet interface with a pluggable backend implementation allowing users to create kubernetes nodes without running the kubelet. This allows users to schedule kubernetes workloads on nodes that aren't running Kubernetes.

## External to a Kubernetes cluster

1. 选择要部署的 Provider，将 virtual-kubelet 一起进行编译。这带来一个坏处，当 provider A 的代码因为某种原因编译不通过，会导致 providor B 的新 feature 也无法正常使用。
2. 在任一Node上执行 `virtual-kubelet --provider aws`，注意该Node没有部署在Kubernetes集群中。你需要按照 aws providor 的要求进行配置，目的是让负载转发到AWS上。
3. run `kubectl get nodes` and you should see a `virtual-kubelet` n   ode

## On a Kubernetes cluster

1. 目前只支持 Must be one of [minikube,docker-for-desktop,docker-desktop]
2. providor 为 mock providor
3. 执行`make skaffold`，可以将pod部署到Kubernetes集群中。
4. run `kubectl get nodes` and you should see a `virtual-kubelet` n   ode

```yaml
spec:
  containers:
  - args:
    - /virtual-kubelet
    - --nodename
    - vkubelet-mock-0
    - --provider
    - mock
    - --provider-config
    - /vkubelet-mock-0-cfg.json
    - --startup-timeout
    - 60s
    - --klog.v
    - "2"
    - --klog.logtostderr
    - --log-level
    - debug
```

### 两个优秀的 kubectl 辅助工具

1. kubectx
2. kubens
