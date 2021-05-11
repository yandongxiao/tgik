# 028: Exploring CockroachDB on Kubernetes

## 一周大事件

1. 《How to deploy web application on k8s with contour and  cert manager》
2. 《On securing the kubernetes dashboard》
3. k8s graduated from CNCF
4. 《Debugging “From Scratch” on kubernetes》



## StatefulSet

1. 作者先定义了什么 Stateful Workload：希望使用持久化存储（不是指数据库系统，而是文件系统上的文件）来保存状态的应用。
2. 一般使用PV的模式是ReadWriteOnce。
   1. 只能有一个Pod来挂载该PV，不允许多个Pod同时挂载。GCP可以，AKS不允许。
   2. Pod内，你可能是可以并发读写的，这与 Kubernetes Provider 的实现有关。
3. StatefulSet 与 Deployment 类似，支持滚动更新，但是它没有 ReplicaSet 这一层。
4. StatefulSet 被删除时，对应的 PVC 对象不会被删除。

## CockroachDB

1. 该DB是基于 Google 的 Spinner 实现的。开发也主要来自谷歌
2. 目前，有 20K Start。支持事务。支持OLTP场景，同时支持轻量级OLAP场景。 
3. CockroachDB is a cloud-native SQL database。
   1. 它是构建在 Kubernetes 之上的数据库系统，充分利用了 Kubernetes的基础能力，比如 StatefulSet
   2. 所以，它没有什么Operator。与之对应的是，Mysql，Cassandra 系统如果部署到Kubernetes上，需要做一些改造和适配，形成一个Operator。

## Read Manifest

1. YAML文件的注释很详细，值得称赞。

2. 在所有的对象中，没有指定命名空间

3. Role 和 CulsterRole 权限详细

   1. 它们同名，但是是不同类型的对象，都是 cockroachdb。
   2. 详细的标准：从 Role 和 ClusterRole 的权限设置上，能推断出 cockroachDB 做了什么事情。

4. ClusterIP 类型的 Service 

   1. 详细：可以推断出，客户端的请求会路由到任意一个Pod上，它们都可以做Query

5. Headless 类型的 Service

   1. 详细：方便Pod之间通信，一般搭配着Stateful Workload来使用.
   2. Annotation tolerate-unready-endpoints: 确保通过 DNS SRV方式，可以查看到所有的Pod。即使该Pod not ready.

   ![image-20210504130136014](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210504130136014.png)

6. PodDisruptionBudget 规定了一次可以删除的Pod个数。
7. StatefulSet
   1. 有一个InitContainer。负责创建公私钥对，向API Server发送 CSR 请求。
   2. 有一个 Container。容器之间实现了multiple raft 协议。
   3. no liveness & readiness

## 使用体验

1. 兼容 postgresql ，拥有SQL查询引擎。
2. 漂亮的UI界面。基于Kubernetes有完整的监控体系。

## 其它

1. watch -n 1 kubectl get pod,pvc
2. 签发证书：Kubectl certificate approve 
3. 从 HTTP 切换成 HTTPS 方式。
   1. 当你通过HTTP的方式服务，服务返回301，返回HTTPS的地址。
   2. 301 是一个持久化的 redirect，即browser会下次不会通过HTTP的方式访问服务，而是直接使用HTTPS。
   3. 如果 localhost:8080 又绑定到了其它的服务，该服务只接收HTTP请求。那么就会出现问题。
4. 如果请求的URL地址时localhost，那么从http切换成https的
5. DNS TYPE
   1. ClusterIP
      1. A 记录。通过 service 名称，返回它的IP地址。
      2. SRV 记录。通过 service 名称，返回它的IP地址。
   2. Headless Service
      1. A记录。返回 pod ip 列表。
      2. SRV记录。不但返回pod的IP地址，还会返回pod的DNS记录。

## TiDB VS CockroachDB

TiDB采用了当前比较成熟务实的技术，例如一致性采用Raft，分布式事务实现采用和Percolator一样的模型，单点存储用RocksDB等，提供给用户的SI隔离级别也够用。同时协议与MySQL兼容，提供数据迁移工具，比较适合中国市场。

CockroachDB采用了稍激进的技术，分布式事务基于乐观锁机制，不加锁，不采用单点时钟，而依赖NTP同步，对于冲突比较重的场景不太适用。一致性协议同样使用Raft。隔离级别方面对外提供Serialable和Snapshot Isolation，相对来讲，S用处不是很大。但是花费了不少精力在上面。可以参看 [详解CockroachDB事务处理系统 - 知乎专栏](https://zhuanlan.zhihu.com/p/26908120)。协议兼容pg协议，主打新的应用，不太适合中国市场。