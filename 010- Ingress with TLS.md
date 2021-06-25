# 010: Ingress with TLS

## Kubernetes world in last week？

1. scanner.heptio.com 界面化的工具，用于sonobuoy的部署，输出sonobuoy的诊断报告。
2. K8s 1.8 release
3. Deployment, DaemonSet 从 extensions group 迁移到 apps group
4. 升级是困难的。社区将中心放在了kubeadm上，以此来解决升级过程中的各种问题。使用 kubeadm 可以完成平滑的升级（from 1.7 to 1.8）。

## 介绍Ingress的工作原理

![image-20210426004949570](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210426004949570.png)

1. 用户输入 xxx.example.com 后，首先访问DNS服务，获取对应的IP地址。目前会使用GEO DNS，即根据用户的地理位置，找到一个合适IP地址。
2. 该IP地址一般是一个LB地址。LB分为L3和L4两种。
   1. L4 LB 的典型代表是：AWS的ELB。特点是：LB会在后端新起一个链接。
   2. L3 LB 的典型代表有：AWS 的 NLB 和 GCP的L3LB（基于maglev论文实现）。特点是源地址时用户的IP地址。
3. ELB 根据（地址+端口号），新建连接，发送到 Ingress。

> AWS 在实现 LoadBalancer 类型的Service时，就是创建一个ELB实例。

## 安装并使用 Ingress

1. 参考：github.com/kubernetes/ingress。它由两个Deployment、一个Service、一个RBAC组成
   1. Deployment A：当路由规则不匹配时，将请求路由到Deployment A
   2. Deployment B：部署两个实例，但是通过分布式锁，只有一个实例在工作。
   3. Service：创建ELB，Ingress Controller 即可对外访问。如果我们的Service想被外访问，也可以创建自己的ELB。但是ELB实例越多，花销越大。Ingress帮我们解决了这个问题。
   4. RBAC：所有namespace的get or list: configmap, nodes, pods, endpoints, secrets(跟TLS相关)，ingress（每个服务可以在自己的namespace下创建ingress，controller可以watch得到的）。
2. Ingress的配置

![image-20210426091254341](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210426091254341.png)

## 问题

1. 使用ELB的问题是source ip会丢失。后端Service接收到的请求，X-Real-Ip 中的地址也不是客户端真正的地址！
2. 解决方案-1：ingress repo 内有解决方案，可能是针对AWS ELB的方案。
3. 解决方案-2：搜索关键字 ingress proxy protocol ，尝试解决

# kube-lego

1. [kube-lego](https://github.com/jetstack/kube-lego) is an older Jetstack project for obtaining TLS certificates from Let’s Encrypt
2. Since cert-managers release, kube-lego has been gradually deprecated in favor of this project. 
3. 使用方式：你只需要在Ingress中添加注解 `kubernetes.io/tls-acme:true` && 添加TLS配置（主要是一个TLS的secret）即可。这个Secret会由kube-lego为你自动生成，你只需要指定secret名称即可。

### 工作原理

1. kube-lego 通过List And Watch的机制监听 Ingress，检查配置了 `kubernetes.io/tls-acm：true`的Ingress
2. 在 kube-lego namespace 下创建一个Ingress，它的 host 配置与用户namespace下的ingress对象一样。但是：
   1. backend 指向了 kube-lego service
   2. 拥有path字段
3. 在 kube-lego namespace 下创建一个Ingress，使得TLS相关的请求会路由到 kube-lego
   1. let's encript 会发送请求（很让人confuse的地方，为什么let's encript会主动发送请求）
   2. 由于path更加匹配，所以该请求由kube-lego服务负责处理。
   3. 它将保存secret，并放置到用户的namespace下！

![image-20210426232228854](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210426232228854.png)

![image-20210426232249501](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210426232249501.png)

1. 斯诺登泄露NSA将应用部署到了GCP
2. 当时，谷歌采用了如上的通讯结构，内部以明文传输
3. 在一个DC内部使用明文还行，但是跨DC使用明文是不安全的！