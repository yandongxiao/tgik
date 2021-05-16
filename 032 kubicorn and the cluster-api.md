# 032: kubicorn and the cluster-api

## kubicorn

1. 没有 release 出来稳定的版本
2. 3 年多未更新
3. 转到 kops 上

## Cluster API

### 为什么要构建 Cluster API

1. 参考：https://cluster-api.sigs.k8s.io/ 。由于Kubernetes安装的复杂性，社区出现了很多的安装软件，质量参差不齐，兼容性也成问题，而且它们多数工作重复。于是 SIG Cluster Lifecycle 发起了 kubeadm 项目，作为Kubernetes安装软件的基础依赖。比如，Kubespray, Minikube, kind 都是构建在 kubeadm 之上。
2. 同理，kubernetes 安装成功之后，还面临着一系列的运维工作，比如，机器的监控，集群升级，集群生命周期的管理（集群升级、删除），多集群管理等。
3. SIG Cluster Lifecycle began the Cluster API project as a way to address these gaps by building declarative, Kubernetes-style APIs, that automate cluster creation, configuration, and management.

