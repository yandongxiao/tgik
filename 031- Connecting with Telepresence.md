# 031: Connecting with Telepresence

##  开发模式总结

1. 本地开发、构建、部署。代表工具是 skaffold。
2. 本地开发，同步文件到远端。非常适用于动态类语言，代表工具是ksync。
3. 本地开发，本地部署。本地环境和开发环境可以互访。

## 一周大事件

1. 《introducing the oracle mysql operator》
2. authentication system 在各个公司不一样，在之前的分享中，Joe 使用了OAUTH2_proxy 的方式，为Dashboard提供了认证功能。Joe 希望介绍一个新主题：how to do single sign-on using Dex
3. Joe 强烈推荐了系列博客
   1. Single Sign-On for kubernetes: An Introduction
   2. Single Sign-On for kubernetes: The Command Line Experience
   3. Single Sign-On for kubernetes: Dashboard Experience
4. **Awesome Kubernetes**
5. **VS Code Kubernetes Tools**

## Telepresence (V2版本可用)

### Choosing a proxying method

1. Inject-tcp。不适合Go。
2. VPN-TCP。适合大部分情况，也是默认方法。
3. Container。类似VPN-TCP，只不是在Container中完成，更安全。

### sshfs

It mirrors things from the cluster back on your local machine。sshfs 的目的是为本地应用程序构建运行环境。

## Quick Start

### 可以在本地访问Kubernetes中的服务

1. 创建应用程序A（Deployment & Service）
2. 使用 telepresence 在本地启动容器，
   1. telepresence 在本地实际上启动了两个容器。
      1. telepresence-local:0.81 是另外一个容器，负责与k8s打通 VPN-SSH 隧道。
      2. 一个容器（alpine容器，与A容器无关）是你指定的容器 。加入到 telepresence-local:0.81  的 Network Namespace 当中。
   2. telepresence 在 Kubernetes 环境也创建了一个deployment。
3. 执行 curl 命令，访问 A

### 使用本地服务访问Kubernetes中的数据

1. Joe 在本地启动kuard的前后端服务。This will start a debug node server on `localhost:8081`. It'll proxy all unhandled requests to `localhost:8080`。比如，DNS服务就是Node Server无法处理的请求![image-20210504220951371](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210504220951371.png)

2. 使用telepresence命令重新启动Go服务。

   1. 前端服务还是可以访问后端服务。
   2. 后端服务的DNS功能可用，且显示Kubernetes相关信息。说明后端服务已经在Kubernetes当中了。

   ![image-20210504221401716](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210504221401716.png)