[TOC]

# 006: kubeadm

## Heptio

它是VMware下的一个组织，拥有自己的博客：https://blog.heptio.com/。它有两款开源产品：

### Velero（5.1k）

1. Velero (formerly Heptio Ark) gives you tools to back up and restore your Kubernetes cluster resources and persistent volumes.
2. 简单介绍：https://twitter.com/LachlanEvenson/status/893509442750169088

### sonobuoy(2.2k)

1. Sonobuoy is a diagnostic tool that makes it easier to understand the state of a Kubernetes cluster by running a set of plugins (including [Kubernetes](https://github.com/kubernetes/kubernetes) conformance tests) in an accessible and non-destructive manner.
2. 简单介绍：https://twitter.com/LachlanEvenson/status/893563156324208641

## 创建虚拟机

1.  OS版本：Ubuntu Server 16.04

2. 19:20 构建好了虚拟机

3. Kubernetes 1.7 版本的 kubeadm 有如下缺陷：

   1. master 只支持单点部署
   2. Kubernetes 的升级是手动升级
   3. 以上两个问题正在着手解决


## 安装 Docker

使用 12.6 版本的 Docker，最新版本的 13 与 Kubernetes 有冲突

```shell
# Install docker
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo "deb https://apt.dockerproject.org/repo ubuntu-$(lsb_release -cs) main" > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-engine=1.12.6-0~ubuntu-xenial
```

## 安装 kubeadm & kubelet

以下内容来自 kubernetes 管控

```sh
# Install kubernetes components
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm
```

### 安装

#### master

```
kubeadm init
```

从 `kubeadm init`的输出结果可以kubeadm为我们做了哪些事情：1. 证书；2. static pod；3. 启动 addon

![image-20210424133856156](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210424133856156.png)

1. [Node RBAC] : kubeadm 使用了两个 authorization 插件。
2. Warning： token 的有效期问题，1.7 是永久的有效期，1.8的有效期是24小时
3. 证书相关。kubeadm的一个重要工作是，确保node和API Server之间的通信是安全的。
   1. Generated CA certificate and key: API Server 内置了一个CA（认证中心）组件，所以这是产生CA自身用的证书和私钥
   2. Generated API server certificate and key: 为 API Server 生成证书和私钥。10.96.0.1 是API Server的Cluster IP，172.31.43.244 是 宿主机的Private IP
   3. Generated API server kubelet client certificate and key: API Server 内部组件也会作为客户端，与 API Server 进行通信，所以，客户端需要自己的证书和私钥
   4. Generated front-proxy xxx: 这是为了 API  Aggregation 功能。
4. **/etc/kubernetes/pki 该目录下就存放了上一步生成的各种证书。**
5. API Server serving cert is signed for DNS names .... 这就是API Server可以暴露的DNS名称，如果 API Server 前面有LBS，或者你想给API Server指定一个DNS名称，那么需要借助参数 `--apiserver-cert-extra-sans`。
6. **/etc/kubernetes/manifests** 存放了启动kube-controller-manager, kube-scheduler, kube-apiserver的配置文件。

#### node

```
# 在 master 节点上执行
kubeadm token list

# 在宿主机上执行
kubeadm join --token=xxx api-server-address
```

![image-20210424144524653](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210424144524653.png)

1. 使用 curl --insecure https://172.31.43.244:6443/api/v1/namespaces/kube-public/configmaps/cluster-info 获取集群的信息。
2. 使用 token 信息，验证 cluster info 的正确性。确保 API Server 的合法性。
3. 向API Server 发送请求，创建自己的证书。这时API Server 应该是通过token的方式来验证客户端的正确性。

### 通过本地访问搭建的K8S

你自行搭建的K8S在集群外可能无法直接访问。通过SSH隧道来访问。

```
ssh ubuntu@xxx -L 6443:localhost:6443 
```

