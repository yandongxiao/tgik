# 012: Exploring serverless with Kubeless

## 一周大事件

1. Kubernetes commuinity office hours 上线
2. 作者出了一本书：kubernetes up & running，属于基础类书籍，与 kubernetes in action 类似。
3. 上线 heptiolabs 功能， github.com/heptiolabs/xxx 下面有一些工具：
   1. ktx。 在多集群的环境下，如何管理 kubeconfig 的项目。
   2. eventrouter
4. 作者出的另外一本书：becoming a cloud native organization。站在管理者的角度，介绍如何构建云原生基础架构
5. Kubevirt 一个很有趣的项目，在kubernetes上运行虚拟机。Redhat出品，KVM虚拟化技术

## kubeless简介

1. kubeless.io   6.5K v1.0.8
2. 下载kubeless客户端命令，通过它你可以向k8s上传一段 python 代码。kubeless 负责服务的构建、打包、运行、扩容等工作
3. VS AWS lambda。lambda是一个集成方案，而kubeless更像是一个模块，比如消息队列模块，可以选择kafka，也可以是别的

## 安装

1. 安装客户端命令。brew install kubeless/tap/kubeless
2. 通过apply的方式安装服务。curl -LO http://xx。-L 支持重定向，-O下载文件内存。
3. 作者首先会查看yaml文件，搞清楚他的RBAC规则，组件（Deployment and Service）。猜想 kubeless 是如何工作的。
   1. 你可以利用 kubeless 客户端工具 或者 kubectl 在指定的namespace下创建一个 CR 对象
   2. 该对象创建一些列的资源，包括 Deployment 或 Statefulset、Service、Ingress 等。
4. 遇到的问题
   1. 多个service可以用来做什么？蓝绿发布。
   2. delete statefulset 的时候，不会删除对应的 pvc 对象。
   3. 在 StorageClass 的 Annotation 添加字段，表明它是default storage class。注意该字段的Key，与平台有关，与K8S版本有关

## 运行

1. 通过 kubeless 客户端命令，提交一个任务。
2. 你可以选择使用HTTP、消息队里（如kafka）、定时任务来处理用户的任务。
3. spec.runtime 字段表明了 kubeless 的应用范围。比如是否支持Java？是否支持Python 3.5？如果需要的library不存在怎么办？是否能根据GPU来调度Pod等？因为代码是提交到K8S对象中的，如果代码量很大，超过了K8S对象的大小限制怎么办？

