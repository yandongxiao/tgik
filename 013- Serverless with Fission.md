# 013: Serverless with Fission

## 关于分享

1. 作者在一个创业公司（startup），帮助个人或者公司，更容易地使用Kubernetes。
2. 每周五会有一个分享

## 一周大事件

1. GCP（Google Cloud Platform）的官网上开源了一个 grafeas。它的作用是：audit and govern your software supply chain with grafeas。比如，你可能有检查容器镜像是否安全？是否使用了Root来启动等担忧。
2. blog.docker.com 宣布支持 kubernetes。在 docker 的生态下。
3. github.com/docker/xxx 有很多有趣的项目

## Fission 简介（6.1K）

### NATS

1. NATS 是一个用Go开发的消息队列（message queue）系统。
2. 目前有9.2k的star，版本是v2.2.2
3. NATS 使用的是内存作为消息的存储介质。NATS Stream 是对消息进行持久化了的系统，可以对标Kafka。它的高可用部署，在作者分享时，还是有问题的。

### Influx

1. Influx 是一个时间序列数据库，可以用来存放logs

### YAML文件

1. values.yaml 文件的注释要非常详细，方便人们阅读
2. Service Type 如果是LoadBalancer，意味着：
   1. 它不需要借助Ingress来对外暴露服务。
   2. Kubernetes Service Provider 会创建一个EBS实例，并绑定到该 service。
   3. 该EBS的DNS名称会很长，你还需要做一个CNAME。
3. 组件包括：controller, router, pool manager, buildermgr, kubewatcher, inflxdb, log daemonset，timer, NATS, mqtrigger 等

### 运行

1. build on kubernetes VS intergrate kubernetes 的一个例子。使用 fission 客户端创建的 pod 并没有在 default namespace 下。那么意味着你后续无法实现多租户的功能，这个属于后者（不推荐）。
2. Fission 对外暴露了三个服务：fission router, fission controller, nats-stream。这三个组件不需要任何验证即可访问，这在生产环境是一个很大的问题。尤其是这些组件会以root身份运行。
3. Fission 采用的是将所有的CR放在一个 namespace 下，对应的Pod也放在同一个namespace下。

### 关于启动（100ms级别的启动时间）

1. 你需要在kubernetes 和 CRD 之间，有一个中间层，它们是准备好的 Pod。
2. 这些 Pod 的 security context 可能更开放
3. 热启动会占用一些资源，它的策略会直接影响到管理员对内存、CPU的管理。
4. Pod是namespace级别的，所以Fission 将业务的Pod都放在了一个Namespace下。