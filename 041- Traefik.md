# 041: Traefik

## 什么是traefik?

官方定义如下：Traefik (pronounced *traffic*) is a modern HTTP reverse proxy and load balancer that makes deploying microservices easy. 

## 它有什么特点？和Ingress Controller的区别在哪里？

1. Continuously updates its configuration (No restarts!) 对于 Nginx Ingress Controller 来说，某些时候可能还是要重启服务。
2. Provides HTTPS to your microservices by leveraging [Let's Encrypt](https://letsencrypt.org/) (wildcard certificates support)。大部分的 Ingress Controller 可能不太关心 HTTPS 部分。
3. GO语言编写。
4. 支持多种平台，例如 Docker Swarm、Kubernetes、Mesos。根据平台的不同，Traefik listens to your service registry/orchestrator API and instantly generates the routes

> 注意：与 nginx ingress controller 一样，traefik 也是一个 Ingress controller. 通过 ingress 来使用 traefik



## 对比

https://medium.com/flant-com/comparing-ingress-controllers-for-kubernetes-9b397483b46b

1. Nginx Ingress Controller 应该是官方版本。Official Kubernetes’ Ingress is easy to use, mature and offers nice features which are enough for the most cases.
2. Kong的特点就是插件多。Kong has the richest set of plug-ins (and opportunities they provide) 
3. 貌似对traefik评价不高。Take a look at Traefik and HAProxy if there are increased demands for balancing and authorization methods.
4. Contour has been around for a couple of years, but it still looks young and has mostly basic features built on top of Envoy.

![image-20210517004817312](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210517004817312.png)