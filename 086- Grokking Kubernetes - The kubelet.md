[TOC]

# 086: Grokking Kubernetes - The kubelet

## 简介

1. 27:00 开始相关内容
2. 作者在熟悉 Kubernetes 时，由于时间的限制，主要将精力放在了如何部署、配置 Kubernetes，以及各个组件之间是如何工作的。
3. kube-controller-manager，kube-apiserver，kube-proxy，kube-scheduler，kubelet 都是一个二进制，每次发布版本，都会更新它们的命令行选项。
4. relnotes.k8s.io 记录了每个组件，在每次 release 过程中，修改的内容。
5. 以下内容均来自官方文档：https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/ 

## kubelet 执行 pod 的三种方式

1. 启动 kubelet 时，指定 PATH，启动 static pod
2. 通过 kubelet 暴露的 HTTP 端口，创建 Pod
3. kubelet 从 API Server 拉取并运行 Pod 

## Pod Lifecycle Event Generator（PLEG）

1. kubelet 通过 Control Loop 模式，调用 Container Runtime Interface，监听由 kubelet 创建的容器的状态（实际状态）。
2. kubelet 会上报Pod的状态到 api server。**PLEG 模块负责获取容器状态，并将容器状态转换成 Pod 状态。**
3. 另一方面，kubelet 根据 Pod Manifest（期望状态） 执行相关动作，如 重建容器（调谐）。

## 部署一个 Kubernetes 集群，CNI 插件未安装

```yaml
# kind.yaml 文件
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

# 创建 Kubernetes 集群
kind create cluster --config ./kind.yaml --name kubelet

# 因为没有安装 CNI 插件，`kubectl get nodes` 的状态是 NotReady。
➜  kind k get nodes -owide
NAME                    STATUS     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kubelet-control-plane   NotReady   master   91s   v1.18.2   172.18.0.2    <none>        Ubuntu 19.10   5.10.25-linuxkit   containerd://1.3.3-14-g449e9269
kubelet-worker          NotReady   <none>   54s   v1.18.2   172.18.0.3    <none>        Ubuntu 19.10   5.10.25-linuxkit   containerd://1.3.3-14-g449e9269
kubelet-worker2         NotReady   <none>   56s   v1.18.2   172.18.0.4    <none>        Ubuntu 19.10   5.10.25-linuxkit   containerd://1.3.3-14-g449e9269

# 在 Kubernetes 安装后，`kubectl get pods -A` 输出了默认运行的组件，包括 `etcd`，`kube-apiserver`，`kube-controller-manager`，`kube-proxy`，`kube-scheduler`，`core-dns`。
# coredns 采用的是 Deployment 部署方式
# kube-proxy 采用的是 DaemonSet 的部署方式
# core-dns 相关的 Pod 处于 pending 状态，其它的 Pod 处于 Running 状态。为什么是这样？通过 `kubectl get pods -A -o wide`，`kubectl get nodes -o wide` 比较发现，处于 running 状态的 pod 使用了 hostNetwork: true 特性，即使用了宿主机的网络。
# 所以，kubelet 和 api-server 都采用的是宿主机网络，它们的交互不依赖 CNI 插件的。
➜  kind k get pods -A -owide
NAMESPACE            NAME                                            READY   STATUS    RESTARTS   AGE   IP           NODE                    NOMINATED NODE   READINESS GATES
kube-system          coredns-66bff467f8-kgspw                        0/1     Pending   0          90s   <none>       <none>                  <none>           <none>
kube-system          coredns-66bff467f8-p24vg                        0/1     Pending   0          90s   <none>       <none>                  <none>           <none>
kube-system          etcd-kubelet-control-plane                      1/1     Running   0          98s   172.18.0.2   kubelet-control-plane   <none>           <none>
kube-system          kube-apiserver-kubelet-control-plane            1/1     Running   0          98s   172.18.0.2   kubelet-control-plane   <none>           <none>
kube-system          kube-controller-manager-kubelet-control-plane   1/1     Running   0          98s   172.18.0.2   kubelet-control-plane   <none>           <none>
kube-system          kube-proxy-cfpvc                                1/1     Running   0          72s   172.18.0.3   kubelet-worker          <none>           <none>
kube-system          kube-proxy-gsm8l                                1/1     Running   0          74s   172.18.0.4   kubelet-worker2         <none>           <none>
kube-system          kube-proxy-pbgrk                                1/1     Running   0          89s   172.18.0.2   kubelet-control-plane   <none>           <none>
kube-system          kube-scheduler-kubelet-control-plane            1/1     Running   0          98s   172.18.0.2   kubelet-control-plane   <none>           <none>
local-path-storage   local-path-provisioner-bd4bb6b75-5nkmh          0/1     Pending   0          90s   <none>       <none>                  <none>           <none>
```

## 从 API Server 查看 Kubelet 的信息

```bash
# 从 API Server端，查看 kubelet 的信息
# 这是请求 API Server 的URI：/api/v1/nodes/kubelet-worker
# API Server 会将 proxy 后面的 URI 路由到 kubelet
# 这是查看 kubelet 配置的一种方式
kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/configz

# 查看 /metrics 接口
kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/metrics

# 查看 /healthz 接口
kubectl get --raw /api/v1/nodes/kubelet-worker/proxy/healthz
```

## 查看 kubelet  的配置

```shell
# 查看 kubelet 的启动命令，主要是 kubelet 的配置文件的位置
root@kubelet-control-plane:/# systemctl cat kubelet
root@kubelet-worker:/# systemctl cat kubelet
# /kind/systemd/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=http://kubernetes.io/docs/

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
# NOTE: kind deviates from upstream here with a lower RestartSecuse
RestartSec=1s

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS

# KUBELET_KUBECONFIG_ARGS
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"

# KUBELET_CONFIG_ARGS
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"

# KUBELET_KUBEADM_ARGS 暂时忽略

# KUBELET_EXTRA_ARGS
EnvironmentFile=-/etc/default/kubelet

# 你自定义的kubelet参数，可以放在这个变量指定的位置。
root@kubelet-worker:/# cat /etc/default/kubelet
KUBELET_EXTRA_ARGS=--fail-swap-on=false

# music: /etc/default/kubelet
root@pri1-music-k8s3:/home/yandongxiao# cat /etc/default/kubelet
KUBELET_EXTRA_ARGS="--allowed-unsafe-sysctls=kernel.shm*,kernel.msg*,kernel.sem,fs.mqueue.*,net.* \
  --node-ip 10.239.83.68 \
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --cni-bin-dir=/opt/cni/bin \
  --cni-conf-dir=/etc/cni/net.d \
  # 配置文件
  --config=/var/lib/kubelet/config.yaml \
  --network-plugin=cni  \
  # 日志输出目录
  --log-dir=/data/log/kubelet \
  --logtostderr=false \
  --pod-infra-container-image=harbor.cloud.netease.com/qzprod-k8s/pause-amd64:3.1 \
  --register-with-taints node.netease.com/phase=checking:NoSchedule \
  --node-labels node.kubernetes.io/instance-type=bare-metal,network.netease.com/kube-proxy=lb-masquerade,topology.kubernetes.io/zone=,node.netease.com/phase=checking \
  # 打印更多的日志
  -v=4"
```

## 查看更多的 kubelet 日志

```bash
# 查看日志
# journalctl -flu kubelet 的日志并非 kubelet 的全部日志
root@kubelet-control-plane:/# journalctl -flu kubelet

# 作者决定在 KUBELET_EXTRA_ARGS 中添加配置
root@kubelet-control-plane:/var/lib/kubelet# echo 'KUBELET_EXTRA_ARGS=--fail-swap-on=false -v 3' >/etc/default/kubelet

# 重启 kubelet
root@kubelet-control-plane:/var/lib/kubelet# systemctl daemon-reload
root@kubelet-control-plane:/var/lib/kubelet# systemctl restart kubelet

# 注意查看 kubelet 全量日志的方式：journalctl -u !
root@kubelet-control-plane:/# journalctl -flu kubelet
```

## 重启 kubelet

```bash
# 重启 kubelet
# taking changed configurations from filesystem and regenerating dependency trees
root@kubelet-control-plane:/var/lib/kubelet# systemctl daemon-reload
root@kubelet-control-plane:/var/lib/kubelet# systemctl restart kubelet
```

## 是否可以删除 Static Pod?

作者开了两个窗口，在每个窗口执行如下的命令。

- 从 api server 的角度来看，确实是可以删除的。

```bash
# etcdclient-kubelet-worker 是直接以Pod的方式启动，没有OwnerReference。
# 参见下文，修改 image 为国内源
➜  ~ kubectl -n kube-system delete pod etcdclient-kubelet-worker
pod "etcdclient-kubelet-worker" deleted
➜  ~ kubectl -n kube-system get pods
NAME                                            READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-h4x9x                         0/1     Pending   0          2d12h
coredns-74ff55c5b-qhg4d                         0/1     Pending   0          2d12h
etcd-kubelet-control-plane                      1/1     Running   1          33m
# 删除后重建的 pod，age == 1s
etcdclient-kubelet-worker                       0/1     Pending   0          1s
kube-apiserver-kubelet-control-plane            1/1     Running   1          33m
kube-controller-manager-kubelet-control-plane   1/1     Running   2          2d12h
```

- 从 kubelet 的角度看，删除动作从来没有发生过。CREATED 没有发生改变

```bash
# container runtime interface
# crictl 与 docker 命令类似
root@kubelet-worker:/etc/kubernetes/manifests# crictl ps   
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
e8414ae114439       2c4adeb21b4ff       16 minutes ago      Running             etcdclient          0                   65c006a99a3d0
e932c79f6955e       23b52beab6e55       37 minutes ago      Running             kube-proxy          2                   2c131d84150ed
```

总结：

1. static pod 完全由 kubelet 来管理，api server 无法删除 static pod

1. adminssion control 的工作模式：在请求 API Server 创建 Pod 对象之前，admission control 可以限制 pod manifest 的内容。这对于 static pod 来说是无效的。因为kubelet 与 api Server 通信时，报告的不是要创建一个Pod（不是请求），而是说这儿有一个pod（而是通知）。

## 删除 kube-proxy 相关的 pod

```bash
# delete pod，因为不是 static pod, 所以可以删除成功
# 重建出来的是：kube-proxy-hhg2l
k -n kube-system delete pod kube-proxy-rj6zk

# 查看日志，注意要给 kubelet 添加 -v 参数，以获取更多的日志。
root@kubelet-control-plane:/# journalctl -u kubelet | grep -i kube-proxy

# 日志内容
# 通过查看事件以及 kube-proxy 的控制逻辑，了解 kube-proxy 的主要作用
journalctl -u kubelet | grep -i kube-proxy | grep 'Event(v1.ObjectRefere'
```

![image-20210715173246126](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210715173246126.png)

## 通过 etcd client 访问 ECTD Server

1. 在**管控节点**，通过 static pod 方式安装 etcd client，`curl -LO git.io/etcdclient.yaml`。etcd client 的配置信息以 `env` 的方式配置在 pod manifest 中。
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

```bash
# rp_filter 不对网络包的原地址进行校验
# cheat xargs 查看它的用法
kind get nodes | xargs -n1 -I {} docker exec {} sysctl -w net.ipv4.conf.all.rp_filter=0

# 在 calico 的官网，下载插件
curl https://docs.projectcalico.org/manifests/calico.yaml -LO
kubectl apply -f calico.yaml

➜  personal git:(master) ✗ kubectl get nodes
NAME                    STATUS   ROLES    AGE    VERSION
kubelet-control-plane   Ready    master   149m   v1.18.2
kubelet-worker          Ready    <none>   148m   v1.18.2
kubelet-worker2         Ready    <none>   148m   v1.18.2
```

### kubectl exec 的背后原理

1. Kubectl exec 发送 HTTPS 请求到 API Server
2. API Server 作为一个七层代理，对用户的身份和权限进行校验
3. 最后，**API Server 自己作为客户端**，向 kubelet 转发请求。

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
    # 在执行 kubeadm join 时，通过 CSR(Certificate Signing Request) 获取 CST(certificate的缩写) 。也称为 kubelet client key。一般应用于 kubelet 作为客户端，请求 api server 时，kubelete 的认证信息
    "tlsCertFile": "/var/lib/kubelet/pki/kubelet.crt",
    "tlsPrivateKeyFile": "/var/lib/kubelet/pki/kubelet.key",
		# 什么是rotateCertificates？Certificate rotation (which means the replacement of existing certificates with new ones) is needed when:
		# Any certificate expires.
		# A new CA authority is substituted for the old; thus requiring a replacement root certificate for the cluster.
		# New or modified constraints need to be imposed on one or more certificates.
    "rotateCertificates": true,
    
    # kubelet 作为 server 时，需要认证客户端的身份
    # 有两种方式：
    # 1. 通过 CST 的方式访问 kubelet, kubelet 借助 CA 来认证客户端身份；
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

1. 在 master 所在的宿主机上，通过查看 `/kind/kubeadm.conf`，找到了 container runtime 的配置。

2. 作者认为，kubeadm 应该可以自动地判断出 node 上的 container runtime的实现，从而自己做出配置 container runtime 的选择。

3. 下面的截图说明：kubelet 通过 CRI 协议与 container runtime 进行交互，所以在 container runtime 的配置中，有 --node-ip, --container-runtime 等参数。

   ![image-20210530150042056](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210530150042056.png)

4. 你可以通过 crictl ，与 container runtime 进行交互。与 docker client 不同的是，它会包含一些 pod 相关的信息。

5. 这是 kubeadm 使用的一个 annotation，表示该节点使用 containerd 作为 contaienr runtime。

   ![image-20210530145216166](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210530145216166.png)
