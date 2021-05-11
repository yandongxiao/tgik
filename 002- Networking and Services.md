# 002: Networking and Services

## Networking

### Pod和Node拥有独立的 Network Namespace。

1. Borg系统采用的是共享Network Namespace。这使得它在网络层面上的实现变得简单，但是把负责留给了上层的应用程序。
2. 首先，共享 Network Namepace，导致Pod监听的端口是不可预知的。
3. 其次，在做单元测试的时候，你可能需要临时启动一个Server。如果没有办法预知该Server监听的端口，你就必须通过 Service Discovery System，动态地获取服务监听的端口。这无形中增加了编写单元测试的复杂度。
4. 最后，Borg 需要实现Service Discovery System。

### Pod Network 的特性

1. Pod 与 Node 拥有独立的 Network Namespace
2. Pod 之间拥有独立的Network Namespace
3. Pod之间可以通过 Pod IP 通信
4. Node上存储了所有Pod的路由信息，所以，From Node To Pod 是可以的
5. From Pod To Node 双向通信都是可以的

### Pod 与 Node Not In Cluster

1. Pod 可以访问 Node not in cluster
2. 但是 Node Not In Cluster 是不可以访问 Pod 的
3. Calico，Weave 方案都是不支持 Node Not In Cluster 访问的
4. 常见解决方案是：使用 AWS 的路由功能，借助 Load Balance Service。
5. 一种解决方案是：网络方案可以覆盖更多的Node，Kubernetes 所在的Node只是它的一个子集。在Node上安装网络组件，使得Pod可以访问到Node。

### CNI（Common Network Interface）

1. CNI的工作原理：不借助Docker，自行设置 pause 容器的Network Namespace，启动容器，并加入之前准备好的Network Namespace。
2. CNI的配置目录：/etc/cni/net.d
3. 当使用Kubeadm安装Kubernetes以后，在它的/opt/cni/bin目录下，保存了常见的网络插件

## Service

### Service 类型

ClusterIP、ExternalName、LoadBalancer、NodePort、None

### ExternalName

该服务部署在集群外，你通过ExternalName类型的Service，给它起了一个别名

### None

1. 给 service 的 Label selector 起了一个别名
2. 创建了 Endpoint 对象，相当于提供了 Endpoint API 接口

### ClusterIP

1. ClusterIP 类型的Service，获得了一个虚拟IP。
2. 该虚拟IP的实现，依赖于kube-proxy，它是以DaemonSet的方式运行在每个Node上。
3. Kube-proxy 通过监听Service和Endpoint对象，更新Node上的iptables规则
4. DNS不适合在构建在Pod上，因为Pod IP会时刻变化。Kubernetes 则是将DNS构建在了Service之上。所以，有 kuard.default 域名。

### NodePort

1. 在所有Node上暴露一个统一的端口号，例如，23231。
2. 借助 kube-proxy，实现将对宿主机的23231访问，路由到Pod中

### LoadBalancer

1. cloud provider 需要支持
2. 在 Service Status 字段部分，会生成一个LB地址。
3. 当你在集群外访问LB地址时，它会被路由到任一Node的那个端口号上，比如，32321。

