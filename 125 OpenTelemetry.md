[TOC]

# TGI Kubernetes 125: OpenTelemetry （jaeger）



## 什么是分布式追踪技术

1. 分布式追踪由两部分组成：客户端和服务端。
2. 应用程序如果想使用分布式追踪技术，就需要使用客户端进行埋点（instrumentation）。客户端使用 OpenCensus 协议或者 Opentracing 协议，与服务端通信。
3. 服务端对数据进行收集，分析，展示。

> 分布式追踪技术的实现：zipkin, jarger

## 什么是 OpenTelemetry？

![img](https://opencensus.io/img/logo-sm.svg) OpenCensus is a set of libraries for various languages that allow you to collect application **metrics** and **distributed traces**, then transfer the data to a backend of your choice in real time. 

OpenTracing is comprised of an API specification, frameworks and libraries that have implemented the specification（说的就是OpenTelemetry）, and documentation for the project. OpenTracing allows developers to add instrumentation to their application code using APIs that do not lock them into any one particular product or vendor.

这两种协议在很大程度上的功能重合，于是合并推出了 OpenTelemetry 协议。

## Jaeger 的架构

Jaeger Controller 通过 List and Watch 的方式监听，Jaeger CRD 对象。当 Jaeger Controller 监听到 Jaeger CRD（test） 被创建时，controller 会额外创建如下Manifest：

![image-20210606161616002](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210606161616002.png)

1. jaeger.jaegertracing.io/test  ==> 创建 Deployment 对象  ==> 创建 Replicaset 对象
2. pod/test-c77b75bc9-xxxx ==  jaeger 实例对应的Pod。**对应上图的 jaeger collector。**
3. service/test-query   ==> 通过 Ingress 的方式访问该 jaeger 实例。所以，不同 team 之间使用不同的 jaeger 实例，进行隔离。
4. ingress.extenstions/test-query
5. service/test-agent ==> jaeger-client（语言的SDK） 访问 jaeger-agent （sidecar），发送埋点数据。
6. service/test-collector   ==> jaeger-agent 通过 test-collector 访问 jaeger-collector，推送埋点数据。
7. service/test-collector-headless

## Setting Up an Instrumented App

1. 通过搜索 jaeger exmaple app hot，定位要部署的App
2. hotrod 的原始部署方式是 docker compose，作者将它转换成 k8s yaml （Deployment + Service）
3. Jaeger 通过 Annotation 的方式自动注入 sidecar，Annotation 中需要写清楚，使用了哪个 jaeger 实例。
4. 作者本以为 Deployment 对象的修改是在 mutating webhook 来完成的，但实际却通过 jaeger deployment 的 List and Watch 来完成的。这导致的一个问题是，当 Deployment X 被部署后，由于 Pod Template 会被修改，Pod会被立即删除重建。这不是 Deployment Owner 所希望见到的。
5. 之所以没有使用 mutating webhook 的方式来实现，是因为 operator SDK 当时不支持（2020.07）。

## 术语（trace and span）

![image-20210606164510305](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210606164510305.png)

## 其它

1. Octant 类似 Kubernetes Dashboard。
2. yaml structure for Intellji。当 yaml 文件中，定义了多个对象，通过该插件，可以方便浏览数据。
3. 作者使用 miro.com 在线画图工具。
