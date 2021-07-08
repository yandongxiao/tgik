# 101: Grokking Kubernetes - API Server: Part 2/X

## 简介

1. 28:00 开始介绍 API Server
2. kubectl auth can-i --list 展示可以操作的资源
3. 通过 kubectl 的 --token 参数，可以覆盖当前 kubeconfig file 中的 token。这对于遍历一个 namespace 下的token，结合 kubectl auth，总结它们的权限，是很有帮助的。
4. 使用 certificate 进行认证的时机是在建立 TLS 链接时，使用 --token 则是发生在链接建立以后，发送 HTTP 请求时进行认证。所以，如果 kubeconfig 文件中是 Certificate，那么 --token 是无效的。

## RBAC Authorization

1. 原先的认证方式 ABAC（Attribute Based Access Control）

2. 作者介绍了 Role，ClusterRole，为什么需要Role时，作者给出的一个原因是：对于 namespace 级别的admin，它需要有独立的方式来创建规则，ClusterRole显然会污染到其它Namespace

3. Kubernetes 创建的默认的 ClusterRole 有 admin,cluster-admin,edit,view

   ```bash
   kubectl create sa view
   kubectl create rolebinding view-view-default --cluster-role=view --serviceaccount=default:view
   kubectl get pods --as=system:serviceaccount:default:view
   kubectl auth can-i --list --as=system:serviceaccount:default:view
   ```

4. You can *aggregate* several ClusterRoles into one combined ClusterRole. 背后的原理是，Controller（ClusterRole Aggregation Controller） 动态监听 ClusterRole 对象，动态填充 Aggregation Role 的规则。

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: monitoring
   aggregationRule:
     clusterRoleSelectors:
     - matchLabels:
         rbac.example.com/aggregate-to-monitoring: "true"
   rules: [] # The control plane automatically fills in the rules
   ```

5. audit2rbac 该工具可以根据Kubernetes的审计日志，生成对应的rbac规则

## Admission Controller

1. Admission 是承认和入学的意思，API Server 会向 Admission Controller 发送YAML，Admission Controller 决定是否要执行创建、更新Manifest
2. Admission Controller 和 Admission Webhook的关系？Admission Controller 是 Kubernetes 提供的插件，Admission Webhook 是 Kubernetes 对外提供的插件机制。
3. Admission Controller 提供的插件，有些是默认启动的，有些则不是。如何弄清楚在一个Cluster中，启动了哪些插件？
   1. api server 的命令行启动参数：--enable-admission-plugins，--disable-admission-plugins
   2. Kube-apiserver -h | grep enable-admission-plugins 查看默认安装的插件
4. kubectl api-resources --namespace 展示 CRD 是否是 namespaced 级别。只能查看 Group 信息
5. kubectl api-versions 查看注册的版本。只能查看版本信息
6. kubectl get --raw /apis/batch | json 查看Group、Kind、Version 信息，如果有多个版本，还会写出推荐的版本。

## Postman的作用

1. 通过【Import】【Import From Link】，填写URL地址。
2. Postman 会从该地址（127.0.0.1:8001/openapi/v2）获取所有的接口规范信息。

## Exploring the API

1. kubectl explain
2. kubectl get --raw /

 kubectl explain

 openapi stuffs

 deprecation [docs](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)

​	 [api reference docs](https://kubernetes.io/docs/reference/using-api/api-concepts)

 validating and mutating webhooks [docs](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

 static pods

 

theory of operation

-  access patterns.