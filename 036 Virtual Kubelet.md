[TOC]

# 混合云调度

## 1. Virtual Kubelet（VK）

### 什么是 virtual kubelet？

来自 virtual-kubelet 命令行的解释。virtual-kubelet **implements** the Kubelet interface with **a pluggable backend implementation** allowing users to create kubernetes nodes without running the kubelet. This allows users to schedule kubernetes workloads on nodes that aren't running Kubernetes。

1. 站在 Kubernetes 的控制面板来看，virtual kubelet 实现了 kubelet 的接口，它就是一个普通的node。
2. 站在 virtual kubelet 的角度看，它实现了作为一个Node的接口；同时 virtual kubelet 定义了一套接口，Providor 需要实现这套接口。
3. Provider 负责实现 virtual kubelet 定义的接口。Virtual Kubelet is focused on providing a library that you can consume in your project to build a custom Kubernetes node agent.

![image-20210729145120016](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210729145120016.png)

### VK 目前的状态

1. 3.2K star
2. 项目已经处于稳定状态：v1.5.0 版本；最近一次更新是2个月前。
3. providor: alibaba, Azure, AWS, nomad, huawei

### VK 的应用场景

1. burst to cloud **container as a Service（Caas）** or other **serverless** platforms
2. Bridge to another orchestrator
3. Schedule to loT control planes

> 如果使用 virtual kubelet 的话，用户需要做哪些适配工作？在使用的过程中，哪些操作是不能使用的？

### External to a Kubernetes cluster

#### Mock Provider

```bash
# 下载原代码
mkdir -p ${GOPATH}/src/github.com/virtual-kubelet
cd ${GOPATH}/src/github.com/virtual-kubelet
git clone https://github.com/virtual-kubelet/virtual-kubelet
cd virtual-kubelet

# 构建: 这带来一个坏处，当 provider A 的代码因为某种原因编译不通过，会导致 providor B 的新 feature 也无法正常使用。
# 修改 Makefile 文件，添加如下两个参数
# 使用ossfs方式，传递二进制
GOOS=linux GOARCH=amd64
make build

# 以 mock providor 来启动 virtual kubelet
# https://github.com/virtual-kubelet/virtual-kubelet/tree/master/cmd/virtual-kubelet/internal/provider

# 启动 virtual kubelet
# 常见参数：--provider mock --provider-config /vkubelet-mock-0-cfg.json --startup-timeout 60s --klog.v "2" --klog.logtostderr --log-level debug
virtual-kubelet --nodename vkubelet-mock-0 --provider mock --provider-config ./hack/skaffold/virtual-kubelet/vkubelet-mock-0-cfg.json

{
  "vkubelet-mock-0": {
    "cpu": "2",
    "memory": "32Gi",
    "pods": "128"
  }
}

# apply kubelet
root@debian-0:~# cat /tmp/data
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeName: vkubelet-mock-0
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

# 查看pod信息发现，pod处于running状态，但是无法exec进去。因为本质上，pod并没有启动
root@debian-0:~# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6675fb45bf-zmbfv   1/1     Running   0          12m
root@debian-0:~# kubectl get pods ^C
root@debian-0:~# kubectl exec -it nginx-deployment-6675fb45bf-zmbfv sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
error: unable to upgrade connection: pod does not exist
```

## 2. 基于 Istio 侧重解决服务治理和流量调度

服务之间的流量治理工作，要求已经在多个K8S集群部署了应用。

![image-20210729145224115](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210729145224115.png)

https://www.cnblogs.com/dhcn/p/13398122.html

## 3. Kubernetes federation（[v0.8.1](https://github.com/kubernetes-sigs/kubefed/releases/tag/v0.8.1)，1.8k）

One of the most common scenarios where Federation is desirable is to **scale an application across multiple data centers**.

有专门的SIG：multicluster Special Interest Group。有专门的标准，现在是v2版本。Kubernetes Federation v2 is referred to as “[KubeFed](https://github.com/kubernetes-sigs/kubefed),”；The SIG has a **kubefedctl** CLI tool。

中文文档：https://www.kubernetes.org.cn/5702.html

还在早期探索阶段

# Admiralty

https://github.com/admiraltyio/admiralty/blob/master/README.md#readme

## 参考文档

1. https://virtual-kubelet.io/docs/usage/
