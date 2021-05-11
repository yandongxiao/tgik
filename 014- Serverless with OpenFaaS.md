# 014: Serverless with OpenFaaS

20k



## 一周大事件

1. heptio写了一篇文章：AWS Quick Start for Kubernetes
2. Sonobuoy 是一个针对Kubernetes的诊断工具，通过 scanner 来输出报告
3. Ark 对 kubernetes 的资源进行灾备和恢复
4. Brigade 是微软开源的一个编写脚本的工具，它可以将脚本转换成Pod，并且在多个脚本之间有控制逻辑
5. AKS：微软发布 azure kubernetes service 

## 简介

1. Faas 支持 kubernetes & swarm，目前有2万star，值得关注
2. serverless的工作原理：
   1. 一个 serverless 系统由主要有gateway ，message queue，controller组成。
   2. 工作模式分为：同步请求和异步请求。
      1. 如果是同步请求：首先经过gateway，路由到相应的 function
      2. 如果是异步请求：首先经过gateway，经过 message queue （NATS 或者 Kafka），路由到相应的 function

## 安装（同步）

1. Faas.yaml
   1. no namespace picked。这可能会导致使用者安装到了default namespace 下
   2. gateway 通过 NodePort 的方式暴露了自己的接口
   3. UI 界面和 Gateway 部署在同一个image中。如何方位UI
      1. kubectl port-forward pod。 service, deployment来进行转发，优先选择
      2. kubectl proxy 这其实利用了http proxy来实现，作者不是很推荐
      3. ssh proxy
      4. ELB。通过创建一个LoadBalancer类型的Service，可以为我们创建一个ELB。但是UI直接向全世界暴露，且无需认证！！
   4. 解释为什么 tekton 默认是不提供UI的，且UI无需登录。tekton 希望借助的是 kubeconfig 的认证体系
2. monitoring.yaml
   1. 部署prometheus，alert manager
3. rbac.yaml
   1. 所有命名空间下，需要 service & namespace 权限



## 例子

1. 使用 Build a Serverless Golang Function With OpenFaas
2. 创建函数：./faas-cli new --lang go gohash。本质上是克隆了一个工程到本地。
3. 修改代码。然后使用Faas Cli 工具进行构建、上传等操作。
4. 部署代码。./faas-cli -f gohash.yaml deploy
5. 通过UI工具，发送请求给函数，并拿到执行结果！



