# 005: Pod Params and Probes

### Liveness probe 和 restartPolicy 之间的区别

1. 在 Deployment 中，你只能将 restartPolicy 设置为 Always。
2. 当Pod中的容器退出时，kubelet 会重新创建新的同名容器
3. 当容器出现假死的情况时，kubelet 需要借助 livenessProbe 来探测。

### 同名的 Pod 是否会被调度到其它节点上？

不会。Pod一旦被调度到某节点上，则再也不会被调度到其它节点上。



