# 003: Istio

## History

1. 这一系列的视频的作者是 Joe Beda，CTO，Heptio
2. Istio 并非是凭空想象出来的系统，而是在生产环境下各种问题的解决方法的集大成者。
3. 作者之前并没有使用过 Istio，所以本次分享会从安装Istio开始
4. 作者有一个演讲：**The Operations Dividend**。
5. Istio要解决的问题是微服务之间的通信问题！

## The Operations Dividend

### Different Type Of Services

1. Implementation Detail 级别的服务：应用程序的一部分功能剥离出来，形成一个Library 或 一个 Service。这个 Service 的调用方可能只有一个。
2. Shared Artifact，Private Instance。公司有一个团队负责构建Mysql版本，但是不同的业务会有各自的mysql实例。
3. Shared Instance。服务只有一个实例，所有业务共享。
4. Big-S Service。对组织内部提供服务 VS 对外提供服务是不同的，你需要考虑安全性，流量控制等。

随着微服务的增多，服务之间的治理工作（安全、性能）就成了很多公司逃不掉的问题。比如：

1. 内部服务和外部服务，需要有更多的网络安全保障。
2. 如何判断服务过载？如何定位到该服务？
3. 谁正在访问哪个服务，是否要对此进行流量控制。

这些都是 Istio 要解决的问题。

## Istio 简介

1. 为什么不适用library的方式，而是用side car的方式？
2. 地址：istio.io。
3. 主要贡献者：Google、IBM、RedHat。
4. 主要功能：智能路由和负载均衡；side car 模式；集中化的配置；分布式追踪。

## 安装

1. 