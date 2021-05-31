# 090: Grokking Kubernetes - kube-proxy

## 简介

1. 34:00 开始介绍 kube-proxy
2. Kube-proxy 实现的两种模式：iptables VS ipvs

## iptables

1. 以 DaemonSet 的方式运行在每个节点上；通过监听 service 对象和 Pod 对象，动态地更新 endpoint 对象和 iptables 规则。
2. 对于 Cluster Service，通信过程类似：podA --> NodeA（修改Destination IP） --> NodeB --> PodB。这个过程中，Destination IP 会被修改。
3. 对于 Node Service，在每个宿主机上暴露一个统一的端口，端口号在 30000 - 32767 之间。与 Cluster Service 的 Iptables 规则基本一致。
4. 对于 Node Service，有两种通信情况：
   1. external node -> NodeA（修改Destination IP 和 Source IP）--> NodeB --> PodB。
   2. external node --> NodeB （修改 Destination IP）--> PodB。这种情况比较特殊，要求：NodeB上运行了 target pod；根据 NodeB上的Iptables规则，概率上，正好路由到PodB。
   3. 你可以查看 "Using Source IP" 查看，在不同类型的Service中，修改 Source IP 的行为。
5. 对于Node Service类型，通过 externalTrafficPolicy: Local 字段，可以保证做到 4.b 的情况。即如果 NodeA 上没有 target pod，在外部host上执行curl NodeA操作，不会有任何响应。通过 curl NodeB，即可操作成功。
6. 在宿主机上，执行 iptables-save | grep nginx 可以查看到部分该 service 相关的 iptables 规则。

```shell
➜  ~ kubectl -n kube-system describe ds kube-proxy
Name:           kube-proxy
Selector:       k8s-app=kube-proxy
Node-Selector:  kubernetes.io/os=linux
Labels:         k8s-app=kube-proxy
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 1
Current Number of Nodes Scheduled: 1
Number of Nodes Scheduled with Up-to-date Pods: 1
Number of Nodes Scheduled with Available Pods: 1
Number of Nodes Misscheduled: 0
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           k8s-app=kube-proxy
  Service Account:  kube-proxy
  Containers:
   kube-proxy:
    Image:      k8s.gcr.io/kube-proxy:v1.20.2
    Port:       <none>
    Host Port:  <none>
    Command:
      /usr/local/bin/kube-proxy
      --config=/var/lib/kube-proxy/config.conf
      --hostname-override=$(NODE_NAME)
    Environment:
      NODE_NAME:   (v1:spec.nodeName)
    Mounts:
			# iptables 只是用户态的接口，实际是将配置下发到内核模块
			# kubelet 可能是第一个启动该内核模块的，而该内核模块的启动，
			# 依赖其它的 modules, 所以需要对 /lib/modules 拥有只读权限
      /lib/modules from lib-modules (ro)
			# xtables.lock 保证 iptables 规则操作的原子性
      /run/xtables.lock from xtables-lock (rw)
      # kube-proxy 的配置
      /var/lib/kube-proxy from kube-proxy (rw)
  Volumes:
   kube-proxy:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      kube-proxy
    Optional:  false
   xtables-lock:
    Type:          HostPath (bare host directory volume)
    Path:          /run/xtables.lock
    HostPathType:  FileOrCreate
   lib-modules:
    Type:               HostPath (bare host directory volume)
    Path:               /lib/modules
    HostPathType:
  Priority Class Name:  system-node-critical
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  2m15s  daemonset-controller  Created pod: kube-proxy-47vpz
```

### IPVS

1. kind config 中可以指定 kube-proxy 使用 iptables 或 ipvs
2. ipvsadm -Ln 查看与 iptables 类似的转发规则
3. 作者认为，目前使用 iptables 的方式，可能与 iptables 的设计初衷是有区别的。ipvs 承担了 iptables 的负载均衡部分，旨在减少iptables的变化。