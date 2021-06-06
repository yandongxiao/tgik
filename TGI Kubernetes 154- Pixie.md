# TGI Kubernetes 154: Pixie

# Pixie Demo Day Talk: What happens after you install the Pixie CLI?

Pixie 是一个新的云原生应用程序的可观察性平台。有了它，开发人员可以通过一个 shell 命令看到他们应用程序的所有指标、事件、日志和追踪。有了 Pixie，你不需要添加度量（instrumentation ）代码，设置临时仪表板，或将数据移出集群，就能看到正在发生的事情。这将为你节省宝贵的时间，这样你就可以致力于建立更好的软件，而不是用更好的方法来监控它。

## 架构

![image-20210606124605534](/Users/dxyan06/Library/Application Support/typora-user-images/image-20210606124605534.png)

1. 控制面板访问数据面有两种方式：通过 Pixie Cloud 或 Direct Data Connection
2. Pixie Cloud 模式下，你需要将集群的数据，上传到 Pixie Cloud 中。

## 数据面架构（Pixie Vizier）

![image-20210606124735975](/Users/dxyan06/Library/Application Support/typora-user-images/image-20210606124735975.png)

1. Pixie Edge Module 以 DaemonSet 的方式部署在每个节点上。它搜集结点的数据，并存储到内存当中
2. Collectors 负责将 Pixie Edge Module 的数据进行过滤并持久化。并能根据趋势，实现汇集数据。实现数据的快速查询。
3. Query Broker：Pixie 负责对脚本进行构建、下发。

## 为什么不需要埋点？

eBPF 是我见过的 Linux 中最神奇的技术，没有之一，已成为 Linux 内核中顶级子模块，从 tcpdump 中用作网络包过滤的经典 cbpf，到成为通用 Linux 内核技术的 eBPF，已经完成华丽蜕变，为应用与神奇的内核打造了一座桥梁，在系统跟踪、观测、性能调优、安全和网络等领域发挥重要的角色。为 Service Mesh 打造了具备 API 感知和安全高效的容器网络方案 Cilium，其底层正是基于 eBPF 技术。

> sysdig 也是采用了类似的技术。

