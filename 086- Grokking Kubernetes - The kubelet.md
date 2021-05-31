# 086: Grokking Kubernetes - The kubelet

## 简介

1. 27:00 开始相关内容
2. 作者在熟悉Kubernetes时，由于时间的限制，主要将精力放在了如何部署、配置 Kubernetes，以及各个组件之间是如何工作的。
3. kube-controller-manager，kube-apiserver，kube-proxy，kube-scheduler，kubelet 都是一个二进制，每次发布版本，都会更新它们的命令行选项。
4. relnotes.k8s.io 记录了每个组件，在每次 release 过程中，修改的内容。
5. 以下内容均来自官方文档：https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/ 

## kubelet 执行 pod 的三种方式

1. 启动 kubelet 时，指定PATH，启动 static pod
2. 通过 kubelet 暴露的 HTTP 端口，创建Pod
3. kubelet 从 API Server 拉取并运行 Pod 

## Pod Lifecycle Event Generator（PLEG）

1. kubectl 通过 Control Loop 模式，通过 Container Runtime Interface，监听由 kubelet 创建的容器的状态（实际状态）。
2. kubelet 会上报容器的状态到 api server
3. 另一方面，kubelet 根据 Pod Manifest（期望状态） 执行相关动作，如，重建容器（调谐）。

## 部署一个 Kubernetes 集群，CNI 插件未安装

### kind.yaml 文件

```yaml
➜  086: Grokking Kubernetes - The kubelet cat kind.yaml
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  # the default CNI will not be installed
  disableDefaultCNI: true
```

### 创建 Kubernetes 集群

```shell

kind create cluster --config ./kind.yaml --name kubelet

# 输出结果如下
➜  086: Grokking Kubernetes - The kubelet k get pods -A
NAMESPACE            NAME                                            READY   STATUS    RESTARTS   AGE
kube-system          coredns-74ff55c5b-h4x9x                         0/1     Pending   0          66s
kube-system          coredns-74ff55c5b-qhg4d                         0/1     Pending   0          66s
kube-system          etcd-kubelet-control-plane                      1/1     Running   0          77s
kube-system          kube-apiserver-kubelet-control-plane            1/1     Running   0          77s
kube-system          kube-controller-manager-kubelet-control-plane   1/1     Running   0          77s
kube-system          kube-proxy-7gw4j                                1/1     Running   0          66s
kube-system          kube-proxy-ftsrn                                1/1     Running   0          49s
kube-system          kube-proxy-g84fx                                1/1     Running   0          48s
kube-system          kube-scheduler-kubelet-control-plane            1/1     Running   0          77s
local-path-storage   local-path-provisioner-78776bfc44-r8jcr         0/1     Pending   0          66s

```

1. 因为没有安装CNI插件，`kubectl get nodes` 的状态是 NotReady
2. 在 Kubernetes 安装后，kubectl get pods -A` 输出了默认运行的组件，包括 etcd，kube-apiserver，kube-controller-manager，kube-proxy，kube-scheduler，core-dns。
3. core-dns 相关的Pod处于 Pending 状态，其它的 Pod 处于 Running 状态。为什么是这样？通过 `kubectl get pods -A -o wide`，`kubectl get nodes -o wide` 比较，发现，处于 running 状态的 pod 使用了 hostNetwork: true 特性！
4. 所以，kubelet 与 api-server 的交互式不依赖 CNI 插件的。

## 查看 kubelet  的配置和日志

```shell
# 查看 kubelet 的启动命令，主要是 kubelet 的配置文件的位置
root@kubelet-control-plane:/# systemctl cat kubelet

# 查看日志
root@kubelet-control-plane:/# journalctl -flu kubelet

# 安装vim
root@kubelet-control-plane:/# apt update
root@kubelet-control-plane:/# apt install vim

# journalctl -flu kubelet 的日志并非 kubelet 的全部日志
# 通过 systemctl cat kubelet 找到 /var/lib/kubelet/config.yaml
# 在该配置文件中，并未发现关于日志的配置！
#
# 找到了 kubelet 的启动参数
# ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --cgroup-root=/kubelet
# 
# 作者通过：https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/ 找到 -v 参数可以使用
#
# 结合上面的信息，作者决定在 KUBELET_EXTRA_ARGS 中添加配置
root@kubelet-control-plane:/var/lib/kubelet# echo 'KUBELET_EXTRA_ARGS=--fail-swap-on=false -v 3' >/etc/default/kubelet
root@kubelet-control-plane:/var/lib/kubelet#
root@kubelet-control-plane:/var/lib/kubelet# systemctl daemon-reload
root@kubelet-control-plane:/var/lib/kubelet# systemctl restart kubelet
#
# 注意查看 kubelet 全量日志的方式：journalctl -u !
root@kubelet-control-plane:/# journalctl -flu kubelet
```

## 是否可以删除 Static Pod?

​	作者开了两个窗口，在每个窗口执行如下的命令。

- 从 api server 的角度来看，确实是可以删除的。

```bash
➜  ~ kubectl -n kube-system delete pod etcdclient-kubelet-worker
pod "etcdclient-kubelet-worker" deleted
➜  ~ kubectl -n kube-system get pods
NAME                                            READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-h4x9x                         0/1     Pending   0          2d12h
coredns-74ff55c5b-qhg4d                         0/1     Pending   0          2d12h
etcd-kubelet-control-plane                      1/1     Running   1          33m
etcdclient-kubelet-worker                       0/1     Pending   0          1s
kube-apiserver-kubelet-control-plane            1/1     Running   1          33m
kube-controller-manager-kubelet-control-plane   1/1     Running   2          2d12h

```

- 从 kubelet 的角度看，删除动作从来没有发生过

```
root@kubelet-worker:/etc/kubernetes/manifests# crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
e8414ae114439       2c4adeb21b4ff       16 minutes ago      Running             etcdclient          0                   65c006a99a3d0
e932c79f6955e       23b52beab6e55       37 minutes ago      Running             kube-proxy          2                   2c131d84150ed
```

总结：

1. static pod 完全由 kubelet 来管理，api server 无法删除 static pod

1. adminssion control 的工作模式：在请求 API Server 创建 Pod 对象之前，admission control 可以限制 pod manifest 的内容。这对于 static pod 来说是无效的。因为kubelet与api Server 通信时，报告的不是要创建一个Pod（不是请求），而是说这儿有一个pod（而是同时）。

## 删除 kube-proxy 相关的 pod

```bash
# delete pod，因为不是 static pod, 所以可以删除成功
kubectl delete pod xxx -n kube-system

# 查看日志
root@kubelet-control-plane:/# journalctl -u kubelet | grep -i kube-proxy

# 日志内容
# 通过查看事件以及 kube-proxy 的控制逻辑，了解 kube-proxy 的主要作用
```

## 通过 etcd client 访问 ECTD Server

1. 在**管控节点**，通过 static pod 方式安装 etcd client，`curl -LO  git.io/etcdclient.yaml`。etcd client 的配置信息以 `env` 的方式配置在 pod manifest 中。
2. 之所以要这样做，是为了绕开 API  Server 的认证体系。
3. 在容器内即可执行 etcdctl 命令。`etcdctl get "" --from-key --keys-only | grep secrets`

```yaml
# curl -LO  git.io/etcdclient.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: etcdclient
    tier: debug
  name: etcdclient
  namespace: kube-system
spec:
  containers:
  - command:
    - /bin/sleep
    - 9999d
    image: registry.aliyuncs.com/google_containers/etcd:3.3.10
    imagePullPolicy: IfNotPresent
    name: etcdclient
    resources: {}
    volumeMounts:
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
      readOnly: true
    - mountPath: /var/lib/etcd
      name: etcd-data
      readOnly: false
    env:
    - name: ETCDCTL_API
      value: "3"
    - name: ETCDCTL_CACERT
      value: /etc/kubernetes/pki/etcd/ca.crt
    - name: ETCDCTL_CERT
      value: /etc/kubernetes/pki/etcd/healthcheck-client.crt
    - name: ETCDCTL_KEY
      value: /etc/kubernetes/pki/etcd/healthcheck-client.key
    - name: ETCDCTL_ENDPOINTS
      value: "https://127.0.0.1:2379"
    - name: ETCDCTL_CLUSTER
      value: "true"
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
```

## kubectl exec, log, cp 的实现是在 kubelet

### 安装网络插件，否则无法安装 nginx as deployment

```
# rp_filter 不对网络包的原地址进行校验
kind get nodes | xargs -n1 -I {} docker exec {} sysctl -w net.ipv4.conf.all.rp_filter=0

# 在 calico 的官网，下载插件
```

### kubectl exec 的背后原理

1. Kubectl exec 发送 HTTPS 请求到 API Server
2. API Server 作为一个七层代理，对用户的身份和权限进行校验
3. 最后，API Server 自己作为客户端，向 kubelet 转发请求。

```shell
# 这是请求 API Server 的URI：/api/v1/nodes/kubelet-worker
# API Server 会将 proxy 后面的 URI 路由到 kubelet
# 这是查看 kubelet 配置的一种方式
kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/configz

# 查看 /metrics 接口
kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/metrics

# 查看 /healthz 接口
kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/healthz
```

### 认证和授权

```yaml
➜  /tmp kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/configz | jq
{
  "kubeletconfig": {
    "enableServer": true,
    "staticPodPath": "/etc/kubernetes/manifests",
    "syncFrequency": "1m0s",
    "fileCheckFrequency": "20s",
    "httpCheckFrequency": "20s",
    "address": "0.0.0.0",
    "port": 10250,
    # 在执行 kubeadm join 时，通过 CSR 获取 CST 。也称为 kubelet client key。一般应用于 kubelet 作为客户端，请求api server 时，kubelete 的认证信息
    "tlsCertFile": "/var/lib/kubelet/pki/kubelet.crt",
    "tlsPrivateKeyFile": "/var/lib/kubelet/pki/kubelet.key",
		# 什么是rotateCertificates？Certificate rotation (which means the replacement of existing certificates with new ones) is needed when:
		# Any certificate expires.
		# A new CA authority is substituted for the old; thus requiring a replacement root certificate for the cluster.
		# New or modified constraints need to be imposed on one or more certificates.
    "rotateCertificates": true,
    
    # kubelet 作为 server 时，需要认证客户端的身份
    # 有两种方式：
    # 1. 通过CST的方式访问kubelet, kubelet 借助CA来认证客户端身份；
    # 2. 客户端通过 service account token 访问 kubelet
    "authentication": {
      "x509": {
        "clientCAFile": "/etc/kubernetes/pki/ca.crt"
      },
      "webhook": {
        "enabled": true,
        "cacheTTL": "2m0s"
      },
      "anonymous": {
        "enabled": false
      }
    },
    # kubelet 支持的权限校验的方式是通过 webhook
    # kubelet 访问 api server，确认 User 是否有 exec, logs 的权限
    "authorization": {
      "mode": "Webhook",
      "webhook": {
        "cacheAuthorizedTTL": "5m0s",
        "cacheUnauthorizedTTL": "30s"
      }
    },
    ...
 }
```

> 该份配置没有 kubelet 作为 server 的证书，说明它是自签名证书。

## kubelet 与 container runtime

1. 在master所在的宿主机上，通过查看 `/kind/kubeadm.conf`，找到了 container runtime 的配置。

2. 作者认为，kubeadm应该可以自动地判断出node上的 container runtime的实现，从而自己做出配置 container runtime 的选择。

3. 下面的截图说明：kubelet 通过 CRI 协议与 container runtime 进行交互，所以关于 container runtime 的配置中，有 --node-ip, --container-runtime 等参数。

   ![image-20210530150042056](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210530150042056.png)

4. 你可以通过 crictl ，与 container runtime 进行交互。与 docker client 不同的是，它会包含一些 pod 相关的信息。

5. 这是 kubeadm 使用的一个 annotation，表示该节点使用 containerd 作为 contaienr runtime。

   ![image-20210530145216166](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210530145216166.png)

## kubelet 与 container network

1. 相关配置选项：--cni-bin-dir, --cni-conf-dir

