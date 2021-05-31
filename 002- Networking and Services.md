# 002: Networking and Services

## Networking

### Pod 和 Node 拥有独立的 Network Namespace。

1. Borg 系统采用的是共享 Network Namespace。这使得它在网络层面上的实现变得简单，但是把负担留给了上层的应用程序。
2. 首先，共享 Network Namepace，导致 Pod 监听的端口是不可预知的。
3. 其次，在做单元测试的时候，你可能需要临时启动一个 Server。如果没有办法预知该Server监听的端口，你就必须通过 Service Discovery System，动态地获取服务监听的端口。这无形中增加了编写单元测试的复杂度。
4. 最后，Borg 需要实现 Service Discovery System。

### Pod Network 的特性

1. Pod 与 Node 拥有独立的 Network Namespace
2. Pod 之间拥有独立的 Network Namespace
3. Pod 之间可以通过 Pod IP 通信
4. Node上存储了所有 Pod 的路由信息，所以，From Node To Pod 是可以的
5. From Pod To Node 通信也是可以的

### Pod 与 Node Not In Cluster

1. Pod 可以访问 Node not in cluster
2. 但是 Node Not In Cluster 是不可以访问 Pod 的
3. Calico，Weave 方案都是不支持 Node Not In Cluster 访问的
4. 常见解决方案是：借助 Load Balance Service，暴露一个域名，Node Not In Cluster 可以通过该域名访问该服务
5. 一种解决方案是：网络方案可以覆盖更多的Node，Kubernetes 所在的 Node 只是它的一个子集。在 Node 上安装网络组件，使得 Pod 可以访问到 Node。

### CNI（Container Network Interface）

1. CNI 的工作原理：不借助 Docker，自行设置 pause 容器的 Network Namespace，启动容器，并加入之前准备好的 Network Namespace。
2. CNI的配置目录：/etc/cni/net.d
3. 当使用 kubeadm 安装 kubernetes 之后，在它的 /opt/cni/bin 目录下，保存了常见的网络插件

## Service

### Service 类型

1. ClusterIP
2. ExternalName
3. None
4. LoadBalancer
5. NodePort

### ExternalName

该服务部署在集群外，你通过 ExternalName 类型的 Service，给它起了一个别名

### None

1. 给 service 的 Label Selector 起了一个别名
2. 创建了 Endpoint 对象，提供了 Endpoint API 接口
3. 对 None Service 的域名进行解析时，直接返回 Pod IP 列表，且该 IP 列表中 Pod IP 的顺序可能是不变的。

### ClusterIP

1. ClusterIP 类型的Service，获得了一个虚拟 IP。
2. 该虚拟IP的实现，依赖于kube-proxy，它是以 DaemonSet 的方式运行在每个 Node 上。
3. kube-proxy 通过监听 Service 和 Endpoint 对象，更新 Node 上的 iptables 规则
4. DNS 不适合在构建在 Pod 上，因为 Pod IP 会时刻变化。Kubernetes 则是将 DNS 构建在了 Service 之上。

### NodePort

1. 在所有Node上暴露一个统一的端口号，例如，23231。
2. 借助 kube-proxy，实现将对宿主机的 23231 访问，路由到 Pod 中

### LoadBalancer

1. **cloud provider 需要支持**
2. 在 Service Status 字段部分，会生成一个 LB 地址。
3. 当你在集群外访问LB地址时，它会被路由到任一Node的那个端口号上，比如，32321。
