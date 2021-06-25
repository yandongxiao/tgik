# 123: Grokking Kubernetes: DNS Part 2

## 简介

1.  22:00 开始介绍 core-dns
2. **/etc/resolve.conf 中 多个 nameserver 的关系？**多个 nameserver 之间的关系是平等的。即，如果请求 nameserver-A 失败，则直接返回解析失败。不会再去请求 nameserver-B。 
3.  service discovery things：dig svr any.any.svc.cluster.local 查看所有 service 的 DNS A 记录

## modifying it via configmap

### 默认配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }    
```

## dns query limit

1. dns name server 为了保护自己，有 rate limit 限制。
2. 如何劫持DNS请求？并进行内部解析？

## sub dns resolver

![image-20210531144641720](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531144641720.png)

1. 一个待验证的问题：tgik.k8s.work 相关的 metrics 信息是否会被收集？

##  Let's get fun with node local dns!

详情参见：https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/