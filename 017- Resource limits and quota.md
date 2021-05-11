# 017: Resource limits and quota

## Role、RoleBinding、ClusterRole、ClusterRoleBinding 的组合关系

| For accessing                                                | Role type to use | Binding type to use |
| :----------------------------------------------------------- | :--------------: | :-----------------: |
| Cluster-level resources （Nodes, PersistentVolumes）         |    CluserRole    | ClusterRoleBinding  |
| Non-resource URLS                                            |   ClusterRole    | ClusterRoleBinding  |
| Namespaced resources in any namespace （and across all namespaces） |   ClusterRole    | ClusterRoleBinding  |
| Namespaced resources in a specific namespace <br />resuing the same ClusterRole in multiple namespaces |   ClusterRole    |     RoleBinding     |
| Namespaced resources in a specific namespace<br />(Role must be defined in each namespace) |       Role       |     RoleBinding     |

## 一周大事件

1. One year of heptio。直播事件是2017.11.18
2. Kubecon 大会从今年开始举行
3. 有一个小伙分析了  《what happens when k8s》。这在面试过程中，应该很有用

## Limit & Quota & LimitRange 要解决的问题

### 从调度端来看

1. 它需要知道请求资源的量，从而找到合适的节点将Pod调度上去。这个资源可以是内置资源，比如是CPU、MEM；或者以插件形式暴露的GPU等。
2. 它需要防止恶意的资源请求（Over Commit）。 LimitRange 不但可以限制最低的Request，最高的Limit，还可以为Pod设置默认的 Request 和 Limit。
3. ResourceQuota 解决 Namespace级别可以使用的CPU、MEM资源的总量。这对于多租户是很重要的。
   1. 它只是限制了Namespace中所有Pod可以使用的CPU、MEM的总量，通过 Admission 来作用。
   2. 如果在apply resource quota 之前，Namespace的资源使用量已经超过了 ResourceQota，原先的Pod不会被驱逐。
   3. 但是，新的Pod也无法被调度上去，因为过不了Admission这一关
4. Pod priority 和 preemption 使得优先级高的Pod会抢占优先级低的Pod的资源。

### 从Node端来看

解决 noisy neighbor 尽量少地影响其它Pod，但是目前的方案不够好。比如：Disk就是所有Pod共享，如果一个Pod疯狂的写磁盘，那么就会对其它Pod造成影响，且目前没有好的解决方案。

在Node端，它将服务分为了三类，Pod的类型可以通过 kube describe 查看到：

1. Best Effort
   1. 没有甚至request，也没有设置limit
   2. 当Node上的内存资源不足时，优先删除Best Effort 类型的资源。
2. burstable
   1. 设置了 request 或 limit。但是 request != limit
   2. 当Pod申请的资源量超过了Limit值，那么就会被杀死重启
3. guarrenteed
   1. 不但设置了CPU，还设置了MEM。而且 request == limit
   2. 当Pod申请的资源量超过了Limit值，那么就会被杀死重启

>  一个 Node 上可以使用的CPI、MEM的量，是kubelet 通过公式计算出来的。你可以可以指定固定值。

## kubectl apply 和 kubectl create 的区别

1. kubectl create 会创建对象，如果对象已经存在，则创建失败。
2. kubectl apply 也会创建对象，但是如果对象已经存在，也会改成更新对象。但是它调用的不是Update接口。
3. kubectl 采用的是 three way merge 方式，它会将 本地对象的值作为base，api server 中存储的最新对象，本地修改的对象，这三个对象进行合并。形成一个patch，提交到 api server。
4. 三方合并使得对象的更新，从串行变成了并行。
5. 应用场景：在HPA的场景下，运维人员和HPA Controller都可能会对Deployment对象修改，通过Three-Way-Merge可以做到双方的修改都有效。

## 如何对Pod的调度进行可视化地验证？

1. 安装 hepster。它会收集node上的kubelet的资源信息（包括，POD和Container），并给到prometheus上。
2. 安装 Dashboard。dashboard 启动时探测到 hepster的存在，应该从它那里获取到了Pod的CPU、MEM信息

## Heapster 被废弃

Heapster 在 1.11 被废弃。因为它不产生数据，只是数据的搬运工。

- CPU内存、HPA指标： 改为 [metrics-server](https://github.com/kubernetes-incubator/metrics-server)。Metrics Server is not meant for non-autoscaling purposes。
- 基础监控：集成到prometheus中，kubelet将metric信息暴露成prometheus需要的格式，使用[Prometheus Operator](https://github.com/coreos/prometheus-operator)
- 事件监控：heptio eventrouter

## Component Config 

1. 目前，对 kubelet 的修改还是很费事的。需要在每个NODE上重启kubelet。
2. 理想的方式是，在API Server上修改 kubelet 的配置。kubelet 检测到内容的修改，主动获取最新配置并自启动。

