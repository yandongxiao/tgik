# 122: Grokking Kubernetes: DNS

## What's DNS

1. 25:00 开始介绍 DNS

2. DNS中，name server 一般监听在 53 端口

3. 查看DNS的方法：host google.com

4. 查看 name server 的方法：cat /etc/resolve.conf . 作者的 nameserver 竟然是 127.0.0.53 ，这是因为他在个人电脑上安装了一个 resolved 的服务，负责缓存域名解析的结果。

5. Resolved 服务的配置：/run/systemd/resolve/resolv.conf ，当 resolved 服务无法在缓存中找到 google.com 域名的IP地址时，它会向 /run/systemd/resolve/resolv.conf 中配置的 name server 发送域名 解析请求。

   ![image-20210531102833206](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531102833206.png)

5. `dig test.metal.k8s.work` 查看域名解析结果

   ![image-20210531103425622](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531103425622.png)

   A 表示是 A 记录，是域名和IP地址列表的映射。265 表示域名解析结果的有效期为 265 秒。本地的 name server 使用它来标识何时过期。

6. 在解析 `test.metal.k8s.work` 之前，你获得了 `metal.k8s.work` 对应的 name server。访问该 name server，获取 `test.metal.k8s.work` 的IP地址。`dig NS metal.k8s.work` 该命令的意思是：告知用户，域名 metal.k8s.work 对应的 name server 是哪些。
7. `dig test.metal.k8s.work @ns-1112.awsdns-11.org`，@后面是一个指定的name server，由它来解析 `test.metal.k8s.work` 的 IP 地址
8. Dns.squish.net 是一个域名解析的网站，它能给出域名解析的详细过程。

### DNS In Docker（libc vs muscl）

1. `docker run -it bash bash`，这是基于 alpine 的容器。有以下特殊性：
   1. `apk update` 
   2. `apk add bind-tools` 安装 dns 相关的工具。
   3. alpine 使用 muscl 而非 libc 作为它的 C 库
   4. **muscl 的 DNS 部分没有缓存能力，即每次域名解析都会访问 /etc/resolve.conf 中的 name server。glibc会有缓存**
2.  并查看它的 `/etc/resolve.conf`文件
3. 发现 nameserver: 8.8.8.8，而宿主机上是 127.0.0.1。这是因为，当 docker 探测到宿主机上 nameserver: 127.0.0.1 配置时，它需要做出修改，因为 127.0.0.1 对于容器进程来说，是毫无意义的。

### DNS IN K8S

1. `kubectl run --it --rm bash bash`，查看它的`/etc/resolve.conf` 。

   ![image-20210531113821733](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531113821733.png)

   对比 docker run 启动的容器

   ![image-20210531113851874](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531113851874.png)

2. `serach` 可以指定多个 search path，它的作用就是，你可以缩短域名为 svc，svc.namespace 等。比如 host kubernetes 时，完整的域名是： kubernetes.default.svc.cluster.local。注意：host 命令默认会使用 search path，dig 命令不会。
3. ndots:5 表示 <service>.<ns>.svc.cluster.local。数字越大，域名解析的速度就会越慢。比如，host kubernetes 在进行域名解析时，最多会执行kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster, kubernetes.default.svc.cluster.local 五次域名解析操作，解析操作会在core-dns中完成。不会将请求转发到宿主机的 name server 上。详情参见：what are [ndots](https://pracucci.com/kubernetes-dns-resolution-ndots-options-and-why-it-may-affect-application-performances.html)
4. dig nginx 会失败，因为 dig 默认不会根据 search path 进行搜索。`dig +search` nginx 是可以的。
5. 有趣点1：dig google.com 返回的 TTL 是 30s，这个值是 coredns 设置的值，并非是 google.com 的 name server 的值。一般情况下，nameserver 会将 TTL 设置为 12h - 24h
6. 有趣点2：dig google.com 返回的A记录中，只有一个IP地址。但是，多次执行 `dig google.com`命令，有时会返回另外一个 IP 地址。为什么IP地址没有保持稳定？

### /etc/nsswitch

![image-20210531123300682](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531123300682.png)

1. 如果使用 glibc 库，根据 /etc/nsswitch 的配置；
2. 首先查看 /etc/hosts 文件
3. 然后，查看 /etc/resolve.conf 中的 DNS  Server

###  /etc/gai.conf

1. gai 是 getaddrinfo 的缩写
2. 默认情况下，IPV6 的配置 优先于 IPV4 的配置。当进行域名解析时，如果 name server 返回的结果中，存在 IPV6 的解析结果，那么优先使用 IPV6 地址，作为结果。

### the tls connection

1. mkcert 在本地生成证书
2. 解析证书内容：`cat /etc/kubernetes/pki/apiserver.crt | openssl x509 -text`
3. 作者想表明的是：当你通过HTTPS访问某个server时，该server的证书中包含了访问它的DNS域名或者IP地址，HTTPS请求中的Host字段必须是证书中指定的某一个。

- 

## what's coredns?

1. 1.16 之后，使用 core-dns 代替 kube-dns

2. 部署方式：deployment + service。

3. service 暴露了三个端口：TCP 53，UDP 53，TCP 9153。通过 service 的定义，可知 9153 端口是用来暴露 metrics 信息的。

4. Cluster IP 是其它 Pod 的 /etc/resolve.conf 中 nameserver 的默认值

5. core-dns 的 dnsPolicy 值是 Default，而非默认值 ClusterFirst。Default 表示 /etc/resolve.conf 的配置来自宿主机。core-dns 依赖宿主机的配置，解析外部域名。

6. ClusterFirstWithHostNet。如果 hostNetwork: true 是 true，该 Pod 可以通过域名的方式访问 宿主机的网络，同时解析 Kubernetes service。/etc/resolve.conf 内容还是 core-dns Service 的IP地址。

   ```bash
   KIND:     Pod
   VERSION:  v1
   
   FIELD:    dnsPolicy <string>
   
   DESCRIPTION:
        Set DNS policy for the pod. Defaults to "ClusterFirst". Valid values are
        'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'. DNS
        parameters given in DNSConfig will be merged with the policy selected with
        DNSPolicy. To have DNS options set along with hostNetwork, you have to
        specify DNS policy explicitly to 'ClusterFirstWithHostNet'.
   ```

   > 注意：static pod 不应该使用 ClusterFirstWithHostNet，因为ClusterFirstWithHostNet 依赖 core-dns，而 static pod 不依赖任何组件

