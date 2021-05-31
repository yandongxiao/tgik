# 099: Grokking Kubernetes - API Server: Part 1/2

## 简介

1. 21:00 开始介绍 API Server

## Access Pattern

![image-20210531174216622](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210531174216622.png)

## 命令行参数

1. --authorization-mode 的默认值是 [AlwaysAllow]，即，如果没有添加RBAC配置，默认配置是总是允许所有操作。
2. --bind-address 0.0.0.0 默认监听宿主机（hostNetwork: true）的所有接口
3. --enable-admission-plugins 作者建议在配置 adminssion plugins时，把default 的 plugins 也写上。方便确认某个plugins是否已经开启。
4. **支持多个ETCD Server，比如，做到 Event 与其它 Object 存储分离**

## auth to etcd

以下是 API Server 的部分配置

![image-20210531210053345](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210531210053345.png)

1. 该配置表明，只有一个ETCD Server，且该 Server 与 API Server在同一个宿主机上
2. ETCD Server 通过 TLS 方式检查 API Server的身份，一旦成功，则允许API Server 的操作。ETCD Server 并没有权限校验相关的功能。
3. API Server 作为 Client，需要有 client.crt 和 client.key。

### Other neat etcd tricks!

1. `kubectl get --raw /metrics | grep ^etcd | grep object` 关于 API Server 的 metrics 信息
2. 通过查看 metrics 信息，可以细粒度的了解 api server 与 kubelet 之间的交互，分析延迟。

## auth to kubelet

![image-20210531211230358](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210531211230358.png)

1. kubectl cp, exec, logs 命令：API Server 作为客户端，访问 kubelet。
2. kubectl proxy 和 kubectl port-forward 可能不需要经过 kubelet，直接访问 pod 即可。
3. API Server 作为 Client，需要有 client.crt 和 client.key。

## How the apiserver handles authentication

### 1. certs

![image-20210531221846941](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210531221846941.png)

1. kubeadm 生成的 admin.conf 中的证书的有效期，默认是一年。以证书形式颁发的Cert，不能够 revoke。
2. 通过对证书进行 base64 -d 解码，openssl x509 -text 解码，可以看到该证书对应的 client name 以及 group name。
3. 你可以通过 RBAC 规则，查看 clusterrolebinding 或者 rolebinding，获取权限信息。

### 2. service account token

1. service account --> secret --> token. 该 token 是 JWT 格式

2. ` echo 'xxx' | jwt decode - ` 获取 token 的详情

   ![image-20210531224233131](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210531224233131.png)

1. kid 相当于 token 的签名信息。
2. 负责给token签名的 key，可以在 api server 的命令行参数中指定。
3. token 不会过期
4. 通过删除 secret，即可快速 revoke token。k8s会自动生成一个新的 secret token。

[Alibaba Cloud with 10,000 Nodes k8s cluster](https://medium.com/@Alibaba_Cloud/how-does-alibaba-ensure-the-performance-of-system-components-in-a-10-000-node-kubernetes-cluster-ff0786cade32)

[how kubectl exec works](https://erkanerol.github.io/post/how-kubectl-exec-works/)