# 022: Nuclio serverless system

## 一周大事件

1. kubernetes 1.9.2 released
2. Contour：支持TLS
3. nginx vs envoy
   1. 性能上肯定是Nginx略胜一筹
   2. Envoy 的性能也不差，杀手锏是配置是API Driven
4. 推荐阅读《Introduction to modern netwok load balancing and proxing》

## 如何评估一个Serverless系统

1. 它能支持多少语言？支持到哪种程度？如下，采用的是 Go Plugin 生成 shared library 的方式来生成镜像。镜像比较小，20MB。

![image-20210503181359750](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210503181359750.png)

2. 支持哪种类型的Trigger？比如，同步的HTTP，异步的NATS、Kafka等。

![image-20210503181555077](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210503181555077.png)

3. 如何将本地的代码，如 helloworld.go，部署到Kubernetes上？
   1. 与现成的 Docker Registry 进行集成？还是直接构建镜像，部署？openfaas 和 nuclio 都是采用的第一种方式。
   2. Docker Registry 方案中，如何进行容器镜像的回收？
4. 是否支持容器镜像的版本化。
5. **serverless 是如何完成动态扩缩容的？** 作者分享的这几个方案，貌似都没有提到这个！

## nuclio

1. 这是一个 Serverless Framework 框架。
2. 你可以在 playground 中了解什么是serverless。
   1. UI 界面做的不错，但是也没有认证
   2. 部署做的事情是：构建镜像，推送镜像。创建 functions CRD 对象。监控 Deployment 对象的 Status 状态，确认Pod部署正常！！
3. 它与Docker Register有一定的集成，

### 直接挂载 docker.sock 的风险

1. 容器中，直接挂载宿主机的 /var/run/docker.sock
2. 这开启了一种直接与 docker daemon 进行通信的窗口。lanch container 将不在 Quoat 中限制。
3. Kubernetes 社区对此确实还没有好的解决方案。推荐方法是，使用指定的Node来完成容器的构建和推送。

### default account

1. 默认 account 权限很小
2. 如果绑定 cluster role。任何一个容器都可以和API Server进行通信，且权限超级大！

