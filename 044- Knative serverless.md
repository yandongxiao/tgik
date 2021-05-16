# 044: Knative serverless

## 什么是 knative?

1. 13:30 开始介绍 knative。
2. 找了一个观看次数最多的视频。knative由三部分组成：Build，Server，Eventing

![image-20210511234256047](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210511234256047.png)

# Introducing Knative: Build, Deploy, and Manage Serverless Workloads on Kubernetes by David Currie

1. knative 不是 Faas，而是一个 Paas 平台。providing a higher level abstraction for developers
2. Eventing 模块负责对接各种事件源，并将它转换成 HTTP 请求，由Serer模块处理。
3. 这不是Knative的模式： 一个请求过来就创建一个Pod，这是Faas的一种模式。

### Service

![image-20210512010245425](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512010245425.png)

1. Helloworld-go 是一个简单的 GO HTTP HelloWorld
2. autoscaling.knative.dev/target: "1"  默认并发超过100，就会新建一个Pod实例
3. autoscaling.knative.dev/minScale: "2" 最少保证两个Pod实例

### 灰度策略

![image-20210512010926303](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512010926303.png)

### **配置Event Source**

![image-20210512011313770](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512011313770.png)

1. 配置 Event Source
2. 将 Event Source 产生的 Event 转换成 HTTP 请求。

![](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512011504989.png)

1. Trigger 负责从 Broker 获取请求，可以过滤请求。
2. 并将它转发到 Service

## 什么是 Istio?

1. istio is a service mesh
2. service mesh manages **communication** between microservices
3. 需要弄清楚 microservice 带来的挑战

## 微服务带来的挑战-1

![image-20210511235802924](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210511235802924.png)

## 微服务带来的挑战-2

![image-20210511235312157](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210511235312157.png)

## 什么是 service mesh？

![image-20210512000118619](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512000118619.png)

![image-20210512000528573](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512000528573.png)

![image-20210512000550159](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512000550159.png)

## Istio 架构

![image-20210512001454495](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512001454495.png)

1. 你无需修改Deployment和Service，可在半路上Istio 
2. Istio 的配置和微服务的配置分离

![image-20210512001815572](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512001815572.png)

1. Virtual Service: How to route you traffic **TO** a given destination
2. Destination Rule: Configure what happens to traffic FOR taht destination. 

## Istio 功能

![image-20210512002136863](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512002136863.png)

1. 不但可以通过CRD，配置路由规则
2. 还可以作为service discovery、CA 存在
3. 主动采集所有Pod的数据

## 数据流转

![image-20210512002806702](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210512002806702.png)

1. 可以代替Nginx Ingress Controller
2. Service 只与同Pod上的 Envoy Proxy 通信

### Knative 的组件有哪些？

![image-20210515102803415](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210515102803415.png)

1. 不同用户有不同的使用方式。 



1. serverless 到底说的是什么？在业界是有争论的。knative 认为自己是一个Paas平台，不仅仅可以做Faas。
2. 

### 这些组件是如何一起工作的？

### Knative 适合哪些工作负载？

### Knative VS kubeless VS openFaas ?

### Knative 的 helloworld 是如何工作的？



