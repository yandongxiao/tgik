# 029: Developing Apps with Ksync（1k active）

## 一周大事件

1. k8s 1.10 release.  port-forward 支持 service, deployment 对象。
2. dynamic kubelet configuration 进入 beta 阶段
3. Apache Spark On K8S
4. Jenkins X For K8S

## Ksync 简介

1. 实现类似于 docker run -v /foo:/bar 中，-v 的效果。
2. 如果是动态类语言，比如python，ruby，那么解锁了一种新的开发模式。在本地修改，立即在K8S中生效。
3. 如果是静态类语言，比如 Go，那么sync的内容应该是二进制。
   1. 方案一：容器进程可能要替换成脚本，监听二进制的变更，杀死子进程，创建新的子进程。
   2. 方案二：获取容器名称和它所在的Node，在Node上执行 `docker restart`。
4. **需要在k8s中，部署DaemonSet类型的组件**。

## 开发在K8S上的开发体验

1. 为什么不要再本地部署应用？
   1. 个人在本地部署的环境，配置很可能过期。这个配置，可能需要你花大力气配置出来。
   2. 应用依赖太多，应用太大，在本地部署不现实。
   3. 不应该依赖CI流程来验证开发的代码。

![image-20210503234433321](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210503234433321.png)