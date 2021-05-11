# 020: Argo workflow system

## 什么是 workflow system？

1. 你有一些列的works要完成。
2. 某些work之间需要串行处理，因为下一个work需要以上一个work的输出，作为它自己的输入。
3. 某些work之间可以并行处理，加快速度。
4. 你会追踪works的进度。
5. 如果某个 work 失败了，你可以快速定位到，可以进行问题排查，可以进行重试。

## workflow 的应用场景

1. CICD
2. MapReduce
3. 与 serverless 进行集成，可以做一些有意义的事情。

## 其它同类型的产品

1. tekton  这是从knative中剥离出来的项目
2. Fission Workflow 是基于它的 Fission serverless 项目而构建的。
   1. Joe 不怎么看好 Fission Workflow。原因在于Fission。
   2. Fission 的一个特点是可以支持Pod的热启动，这要求 Fission 的 pool manager 需要有很大的权限，所有的CRD和Pod都需要在同一个namespace中。扩展性不太好。
3. Brigade
   1. 与 git repo 深度集成，无法用于CICD
   2. 可以动态地、增量地构建 workflow，但是却无法查看workflow的全貌
   3. 支持 trigger。Argo 需要通过 Argo CI 来触发 argo workflow 的执行

## Argo Workflow （8.4k，v3.0.2）

### 关于 CRD

1. 通过CRD描述了 workflow 的组成，通过 Mini language 可以实现 IF/ELSE 逻辑。
2. 作者觉得这种 Mini Language 可能会不受控制，后面会变得复杂起来。

### 关于输入和输出

1. 对于小文件或者小数据，可以通过Pod Annotation来存储。获取Container A的输出并将它写到Pod的Annotation当中，这是由Side Car 容器来完成的。
   1. 你在submit一个任务时，需要使用--serviceaccount指定一个有权限写Annotation的Service Account。
   2. 作者认为这里比较好的一个设计：argo controller 的权限与workflow中pod的权限的分离。用户可以控制argo workflow 的权限。
2. 如何在pod之间处理大文件？
   1. 需要借助一个 Object Storage。了解下 minio（27k）。
      1. 你可以将本地磁盘，通过minio，对外暴露成S3存储
      2. 你可以将块存储，通过minio，对外暴露成S3 存储。
   2. 这是一个通用问题。作者期望的是，每个服务的运行，都已经默认绑定好了一个对应的Object Store，无需进行认证。Borg 是这样做的。
3. PVC 同时只能绑定到一个Pod上。

#### 关于UI

1. 有自己的UI界面。只读。