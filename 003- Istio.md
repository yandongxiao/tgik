# 003: Istio

## 什么是 Istio?

1. istio is a service mesh
2. service mesh manages **communication** between microservices
3. 需要弄清楚 microservice 带来的挑战

## 微服务带来的挑战-1

![image-20210511235802924](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210511235802924.png?lastModify=1621385659)

## 微服务带来的挑战-2

![image-20210511235312157](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210511235312157.png?lastModify=1621385659)

## 什么是 service mesh？

它是一种理念，用户只需要关心业务逻辑。网络方面的非业务逻辑，交由 sidecar 来完成。例如：重试、metric信息、tracing 等

![image-20210512000118619](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512000118619.png?lastModify=1621385659)

![image-20210512000528573](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512000528573.png?lastModify=1621385659)

![image-20210512000550159](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512000550159.png?lastModify=1621385659)

## Istio 架构

![image-20210512001454495](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512001454495.png?lastModify=1621385659)

1. 你无需修改Deployment和Service，可在半路上Istio 
2. Istio 的配置和微服务的配置分离

![image-20210512001815572](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512001815572.png?lastModify=1621385659)

1. Virtual Service: How to route you traffic **TO** a given destination
2. Destination Rule: Configure what happens to traffic FOR taht destination. 

## Istio 功能

![image-20210512002136863](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512002136863.png?lastModify=1621385659)

1. 不但可以通过CRD，配置路由规则
2. 还可以作为service discovery、CA 存在
3. 主动采集所有Pod的数据

## 数据流转

![image-20210512002806702](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210512002806702.png?lastModify=1621385659)

1. 可以代替Nginx Ingress Controller
2. Service 只与同Pod上的 Envoy Proxy 通信

-------

## History

1. 这一系列的视频的作者是 Joe Beda，CTO，Heptio
2. Istio 并非是凭空想象出来的系统，而是在生产环境下各种问题的解决方法的集大成者。
3. 作者之前并没有使用过 Istio，所以本次分享会从安装 Istio 开始
4. 作者有一个演讲：**The Operations Dividend**。
5. **Istio要解决的问题是微服务之间的通信问题！**

## The Operations Dividend

### Different Type Of Services

1. Implementation Detail 级别的服务：应用程序的一部分功能剥离出来，形成一个Library 或 一个 Service。这个 Service 的调用方可能只有一个。

2. Shared Artifact，Private Instance。公司有一个团队负责构建 Mysql 版本，不同的业务会有各自的mysql实例。

3. Shared Instance。服务只有一个实例，所有业务共享。

4. Big-S Service。对组织内部提供服务 VS 对外提供服务是不同的，你需要考虑安全性，流量控制等。

   

总结：随着微服务的增多，服务之间的治理工作（安全、性能）就成了很多公司逃不掉的问题。比如：

1. 内部服务和外部服务，需要有更多的网络安全保障。
2. 如何判断服务过载？如何定位到该服务？
3. 谁正在访问哪个服务，是否要对此进行流量控制。

这些都是 Istio 要解决的问题。

## Istio 简介

1. 为什么不使用 library 的方式，而是用 side car 的方式？ 使用 library 的代价是需要维护多个语言的 library
2. 地址：istio.io。
3. 主要贡献者：Google、IBM、RedHat。
4. 主要功能：智能路由和负载均衡；side car 模式；集中化的配置；分布式追踪；流量灰度
