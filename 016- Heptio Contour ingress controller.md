# 016: Heptio Contour ingress controller

## 一周大事件

1. 《Cloud Native Infrastructure》  由于是2017年出版，貌似并没有更新。
2. Weaveworks/flux 一个基于 gitops 理念的 CD 系统。

## Heptio Contour

1. Load Balancing Proxy HTTP
2. CNCF
3. based on envoy

基于 Envoy 的 Ingress Controller。

## 要解决的问题

1. Nginx 的 reload 功能有点不太好。
2. Nginx Controller 会将所有的 Ingress 配置组织在一起，形成Nginx的配置。
3. 这个 Nginx Config Tree 会很大，难以维护和管理。

Envoy 的两大优势：

1. API Driven
2. 传统的方式是：Nginx Controller 将 Nginx配置下发下去。
3. Envoy 是从 Controller 拉取相应的配置。

## 架构

Init contrainer + envoy + contour

1. Init contrainer 作为 envoy 的基础配置。其中重要一项配置是，从 localhost:port 获取配置。
2. contour 作为 Ingress controller，负责输出 Ingress 配置给Envoy
3. 对外支持 ELB 和 NLB

## ELB VS NLB VS ALB

1. 这是源自AWK的三个负载均衡系统。一开始AWS开发的是ELB，随着后面的发展，推出了 ALB 和 NLB

2. ELB 这是一个四层的负载均衡系统。它接受用户请求并创建一个新的连接来转发请求。所以，Pod查看到的IP不是真正的客户端IP。AWS支持通过X-Forwarded-For Header 将客户端的IP地址转发到upstream server

3. ALB 是 Application Load Balancer：是一个七层的负载均衡系统。类似于 Nginx 和 Envoy

4. NLB 是 Network Load Balancer：是一个工作在三层的负载均衡系统，客户端与upstream server之间直接建立连接，无需修改源IP地址。

   >  这里的 upstream server 其实就是 ingress controller 。

## 部署支持NLB 的 Heptio Contour

1. 需要使用 DaemonSet 的方式部署
2. 使用 HostNetwork 直接使用宿主机的网络