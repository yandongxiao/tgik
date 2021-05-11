# 086: Grokking Kubernetes - The kubelet

## kubelet 执行pod的三种方式

1. 启动 kubelet 时，指定PATH，启动 static pod
2. 通过 kubelet 暴露的 HTTP 端口，创建Pod
3. kubelet 从 API Server 拉取并运行 Pod 

## Pod Lifecycle Event Generator（PLEG）

Pod 相关的一系列事件是由 Kubelet 上报的。

## 部署一个Kubernetes集群，CNI插件未安装

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

>  为什么有些Pod是处于运行状态？为什么有些Pod是处于Pending状态？首先，这些是 Static Pod，其次它们声明了hostNetwork: true

## 查看kubelet的日志

```shell
# 查看启动命令
root@kubelet-control-plane:/# systemctl cat kubelet

# 查看日志
root@kubelet-control-plane:/# journalctl -flu kubelet

# 安装vim
root@kubelet-control-plane:/# apt update
root@kubelet-control-plane:/# apt install vim

# 查看更加详细的日志
# 作者查看 /var/lib/kubelet/config.yaml 文件的配置，并未发现关于日志的配置
# 找到了 kubelet 的启动参数
# ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --cgroup-root=/kubelet
# 
# 作者通过：https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
# 找到 -v 参数可以使用
#
# 结合上面的信息，作者决定在 KUBELET_EXTRA_ARGS 中添加配置
root@kubelet-control-plane:/var/lib/kubelet# echo 'KUBELET_EXTRA_ARGS=--fail-swap-on=false -v 3' >/etc/default/kubelet
root@kubelet-control-plane:/var/lib/kubelet#
root@kubelet-control-plane:/var/lib/kubelet# systemctl daemon-reload
root@kubelet-control-plane:/var/lib/kubelet# systemctl restart kubelet
root@kubelet-control-plane:/# journalctl -flu kubelet
```

## 如何排查pod重启问题

```
# 查看日志
root@kubelet-control-plane:/# journalctl -u kubelet | grep -i kube-proxy

# 通过查看事件 以及 kube-proxy 的控制逻辑，了解 kube-proxy 的主要作用
```

## 通过 etcd client 访问 ECTD Server

1. 在管控节点，安装以下static pod。之所以要这样做，是因为为了绕开API  Server的认证体系。
2. 在容器内即可执行 etcdctl 命令。`etcdctl get "" --from-key --keys-only | grep secrets`

```
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

## 安装网络插件

```
# rp_filter 不对网络包的原地址进行校验
kind get nodes | xargs -n1 -I {} docker exec {} sysctl -w net.ipv4.conf.all.rp_filter=0

# 在 calico 的官网，下载插件
```

## kubectl exec 背后的原理

1. Kubectl exec 请求到 API Server
2. API Server 做一个HTTP层的代理，将请求代理到后端。

```shell
# 这是请求API Server 的URI：/api/v1/nodes/kubelet-worker
# API Server 会将 proxy 后面的 URI 路由到 kubelet
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
    # 在执行 kubeadm join 时，通过CSR获取CST。也称为 kubelet client key。一般应用于kubelet作为客户端，请求api server 时，kubelete 的认证信息
    "tlsCertFile": "/var/lib/kubelet/pki/kubelet.crt",
    "tlsPrivateKeyFile": "/var/lib/kubelet/pki/kubelet.key",
		# 什么是rotateCertificates？Certificate rotation (which means the replacement of existing certificates with new ones) is needed when:
		# Any certificate expires.
		# A new CA authority is substituted for the old; thus requiring a replacement root certificate for the cluster.
		# New or modified constraints need to be imposed on one or more certificates.
    "rotateCertificates": true,
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



## Static Pod

1. static pod 完全由 kubelet 来管理，api server 无权处理
2. adminssion control 对于 static pod 来说是无效的

### 是否可以删除 static pod?

- 从 api server 的角度来看，确实是可以删除的

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

