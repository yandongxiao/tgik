[TOC]

# 099: Grokking Kubernetes - API Server: Part 1/2

1. 21:00 开始介绍 API Server

## Access Pattern

![image-20210531174216622](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531174216622.png)

## 命令行参数

```bash
root@kubelet-control-plane:/etc/kubernetes/manifests# cat kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 172.18.0.2:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    # --bind-address 0.0.0.0 默认监听宿主机（hostNetwork: true）的所有接口。
    # 默认值就是 0.0.0.0
    # advertise 是公布的意思，kubectl get nodes -owide 返回的就是这个IP地址。
    - --advertise-address=172.18.0.2
    - --allow-privileged=true
    # authorization-mode 的默认值是 [AlwaysAllow]，即:
    # 如果没有添加RBAC配置，默认配置是总是允许所有操作。
    - --authorization-mode=Node,RBAC
    # api server 的另一个身份就是 CA （证书颁发机构）
    # openssl x509 -text -noout -in  /etc/kubernetes/pki/ca.crt
    - --client-ca-file=/etc/kubernetes/pki/ca.crt

    # 通过插件的方式，扩展 API Server 的功能。有内置插件的。
    # 作者建议在配置 adminssion plugins时，把 default 的 plugins 也写上。
    # 方便确认某个plugins是否已经开启。
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true    

		# auth to etcd
    # 支持多个ETCD Server，比如，做到 Event 与其它 Object 存储分离
    # ETCD Server 通过 TLS 方式检查 API Server的身份，一旦成功，则允许 API Server 的操作。ETCD Server 并没有权限校验相关的功能。
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    # etcd 也是一个CA。api server 需要一个有etcd-ca签发的证书
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    # etcd-servers 使用的是复数
    # 该配置表明，只有一个ETCD Server，且该 Server 与 API Server在同一个宿主机上
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0

    # auth to kubelet
    # kubectl cp, exec, logs 命令：API Server 作为客户端，访问 kubelet。
    # CA 是 Kubernetes。
    # 证书
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    # 由 RSA 算法得出的私钥
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key

    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    
    # api sercer 自己的证书
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

## kubectl get --raw /metrics

1. `kubectl get --raw /metrics | grep ^etcd` 关于 API Server 的 metrics 信息
2. 通过查看 metrics 信息，可以细粒度的了解 api server 与 kubelet 之间的交互，分析延迟。

## How the apiserver handles authentication（身份认证）

### 1. certs

![image-20210531221846941](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210531221846941.png)

1. kubeadm 生成的 admin.conf 中的证书的有效期，默认是一年。以证书形式颁发，不能够 revoke。
2. 通过对证书进行 base64 -d 解码，openssl x509 -text 解码，可以看到该证书对应的 client name 以及 group name。
3. 你可以通过 RBAC 规则，查看 clusterrolebinding 或者 rolebinding，获取权限信息。

### 2. service account token

1. service account --> secret --> token 该 token 是 JWT 格式

2. ` echo 'xxx' | jwt decode - ` 获取 token 的详情

   ```bash
   ➜  personal git:(master) ✗ echo -n 'eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLW13dDhyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyMDdkOThhMy1iMjQzLTExZWEtODFlMS0wODAwMjdkZTE5YzMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.PuuBI2t6UOyF3jCFzJ8Kom7xSWHBvSwMtw_fLTbLRBw9PI5hQnaq7-A_cgoxl9T0P_oH_TM1pJHOFzVDhMOxax09XF-Fth7WbW5OCnlAEKQc8KEas31GQhD0bhZDDcdFzgwzyo8VpmhNa45Ij_0ohXkknyOtKSXABMIZ4cuZH15-ePqVglyOtCy5DsVdrEQgxkYaxUUkz7glGhyxMHK_iYv8uMil98EMge4BYCjpKpw_xqsxoerXSNH8Z1xkoN1vHPrYT1GRP4P1x87CA2yaFtKhCOZpNjGq7WWrugkpxBYpTbQQDI3ppPHGrMcNm2xg-gqDjfT0mG_vbaGlKih6Kw' | jwt decode -
   
   Token header
   ------------
   {
     "alg": "RS256",
     "kid": ""
   }
   
   Token claims
   ------------
   {
     "iss": "kubernetes/serviceaccount",
     "kubernetes.io/serviceaccount/namespace": "kube-system",
     "kubernetes.io/serviceaccount/secret.name": "default-token-mwt8r",
     "kubernetes.io/serviceaccount/service-account.name": "default",
     "kubernetes.io/serviceaccount/service-account.uid": "207d98a3-b243-11ea-81e1-080027de19c3",
     "sub": "system:serviceaccount:kube-system:default"
   }
   ```

   1. kid 相当于 token 的签名信息。
   2. 负责给 token 签名的 key，可以在 api server 的命令行参数中指定。
   3. token 不会过期
   4. 通过删除 secret，即可快速 revoke token。k8s会自动生成一个新的 secret token。

## [how kubectl exec works](https://erkanerol.github.io/post/how-kubectl-exec-works/)

```bash
# 执行 exec 命令
➜  personal git:(master) ✗ k-cust87-control -n kube-system exec -it cleanlog-grd5q sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.

# 查看到 kubectl 进程
➜  086 git:(master) ✗ ps -ef | grep kubectl
  501 87281 85151   0  9:40下午 ttys000    0:00.14 kubectl --kubeconfig=/Users/dxyan06/go/src/g.hz.netease.com/music-cloud-native/environment/onlinetest-control.conf -n kube-system exec -it cleanlog-grd5q sh

# 连接到API Server（127.0.0.1.61620）。
➜  086 git:(master) ✗ netstat -atnv | grep 87281
tcp4       0      0  10.120.223.90.65464    10.197.116.135.6443    ESTABLISHED 131072 131072  87281      0 0x0102 0x00000028


# API Server接收到的请求（与上面不符）
kubectl -n kube-system logs -f kube-apiserver-kubelet-control-plane --tail 10 | grep POST
I0715 13:04:50.645759       1 httplog.go:90] verb="POST" URI="/api/v1/namespaces/kube-system/pods/kube-apiserver-kubelet-control-plane/exec?command=sh&container=kube-apiserver&stdin=true&stdout=true&tty=true"


# API Server 使用用户的请求参数，填充 PodExecOptions 数据结构
# API Server 调用ExecLocation方法，获取到 node 的 URL 地址
# 通过 pod.spec.NodeName 找到对应的 Node
# 通过 node 找到 GetPreferredNodeAddress，node.Status.DaemonEndpoints.KubeletEndpoint.Port

# 定位目标kubelet
➜  086 git:(master) ✗ k-cust87-control -n kube-system get pods cleanlog-grd5q -o jsonpath={.spec.nodeName}
music-beta-k8s2-node-qz-vm2

➜  086 git:(master) ✗ k-cust82-control -n argocd get pods argocd-redis-ha-haproxy-7d7ddf4f9f-mm2nx -o jsonpath={.spec.nodeName}
pri1-music-k8s6.yq.163.org%
➜  086 git:(master) ✗ k-cust87-control get nodes music-beta-k8s2-node-qz-vm2 -o jsonpath='{.status.daemonEndpoints.kubeletEndpoint}'
map[Port:10250]%
➜  086 git:(master) ✗ k-cust82-control get nodes -owide
```