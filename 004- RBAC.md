[TOC]

# 004: RBAC

## kubeconfig

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 根证书，API Server 充当了CA的角色
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop	# 这个名字可以随便起
  user:
    client-certificate-data: 用户的公钥 
    client-key-data: 用户的私钥
```

## `kubectl get nodes`的交互过程

1. 客户端与 API Server 发送 HTTPS 请求，建立TLS链接
2. API Server 将自己的证书发送给客户端
3. 客户端将证书发送给CA（API Server），请求CA验证证书的合法性。
4. 验证通过后，用户携带自己的公钥发送请求到服务端（API Server）
5. 服务端（也是API Server）将证书发送给CA（API Server），请求CA验证证书的合法性。
6. API Server 根据认证插件，识别出用户的身份。
7. API Server 根据授权插件，识别出用户是否有权限发起该请求。

## 通过 x509 证书，生成kubeconfig

1. pem: 私钥信息；csr: certificate signing request；crt: 由CA签发的证书。
2. 根据 https://www.linkedin.com/pulse/adding-users-quick-start-kubernetes-aws-jakub-scholz 生成证书

1. 生成私钥

   ```bash
   openssl genrsa -out jakub.pem 2048
   ```

2. 生成CSR（certificate signing request）

   ```bash
   # 指定用户名称为 users:jakub, 作者应该使用 jakub 会更好。
   # 指定 Group 信息为 cool-people
   openssl req -new -key jakub.pem -out jakub.csr -subj "/CN=users:jakub/O=cool-people"
   ```

3. 准备CSR请求。

   ```bash
   apiVersion: certificates.k8s.io/v1beta1
   kind: CertificateSigningRequest
   metadata:
     name: user-request-jakub
   spec:
     groups:
     - system:authenticated
     request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZUQ0NBVDBDQVFBd0VERU9NQXdHQTFVRUF3d0ZhbUZyZFdJd2dnRWlNQTBHQ1NxR1NJYjNEUUVCQVFVQQpBNElCRHdBd2dnRUtBb0lCQVFDNGxQMTNVZHlBTDZ1QUttTVROeDU5RUtlTE14VkNWSTJVWUhWL2hvYjdpelBGCldRSmxaY1dEWllnL1dIVTJoMFJ3TGNkeWpaVTVHcTFPdS9IMjQxSWpBL3dTMXJSWnc1c0ZrNG8zYWdBZlJ0QngKbzZDc3czS0Q3eEpvdEtpRURzOXFqbUJPNzFwU1kzK3BqdHBZbUpxWmZ5cmV2Y094Wm12NXpSU2NaWUZkb1ZMdwptb0VxeUx0a1hDNzVpeXBGTFg2WjJwK25POFE2NHFtU2VZWHRjNFZBbjZhbjdlUFJFU3k5S09NUURZTHRUZENBClBaTVpDeXV0WnBXQnFqMXZlV3NkaWxweUlyaXdRYWtNQ2tBdURHbUxGbzJhbFk3REV4Z0t0ay8rZlZSaEkyY1YKRUJzVUptaVBuZ1NrMXhKRzZjWjdUZE12UXYxSnIyaGNsc1NDY1IySkFnTUJBQUdnQURBTkJna3Foa2lHOXcwQgpBUXNGQUFPQ0FRRUFiL1BBcVp0eUovUHA2MWtXcG54azNDek9SNTdhT09JSHU3YkVUN1lmcE9GazFSN0JkV3hWClU1RXUrK3pTNVZmQm9XaVNNekV1eENrVHZsNU1CVGxDL3NvQWprT2YzQTI2aUFpeFJibksrUWw0ZGt1MGJQTisKT2syZ3pqTXZyM1hsT1ExZVZnQytjTVZzeXJpZENUMzRwcHVPNm9RTkZnVk1GblZ3bHozdjJnZkVDK3c1RnVndgpOVXcxT21td0c1RTZKN0VpL2ZSTk1scnUyTzZaUlQvMGYyWWpnSUhwT3RONTJYSDhhc3cvMjhMOWt1c1J6aHR5CnFLa0xxWGhkYW4xSDRsR1E2VUwxd1V4UU8zUWhyRnUrbkhFRTV3SVdDNEFra0N4K1NPQ3RpalNmV2Vhb3dkTzcKRG8zaWJvWGNwTzVSUDYvYUhEWE9kV0ZXc2EvQituVG9Vdz09Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
     usages:
     - digital signature
     - key encipherment
     - client auth
   ```

   ```bash
   # 注意需要Base64编码
   # pbcopy 复制到剪切板
   cat jakub.csr | base64 | tr -d '\n' | pbcopy
   
   # apply
   kubectl apply -f csr.yaml
   ```

4. 签发证书

   ```bash
   kubectl certificate approve user-request-jakub
   ```

5. 获取证书

   ```bash
   kubectl get csr user-request-jakub -o jsonpath='{.status.certificate}' | base64 -d > jakub.crt
   ```

6. 生成 kubeconfig 文件

```bash
# Creating the config file.
# We need to create a new cluster, credentials and context with the certificates we have just generated.
kubectl --kubeconfig ~/.kube/config-jakub config set-cluster jakub --insecure-skip-tls-verify=true --server=https://127.0.0.1:61620
kubectl --kubeconfig ~/.kube/config-jakub config set-credentials jakub --client-certificate=jakub.crt --client-key=jakub.pem --embed-certs=true
kubectl --kubeconfig ~/.kube/config-jakub config set-context jakub --cluster=jakub --user=jakub
kubectl --kubeconfig ~/.kube/config-jakub config use-context jakub
```

7. 创建RBAC规则

   1. 官网：https://kubernetes.io/docs/reference/access-authn-authz/authorization/
   2. htptio的教程：http://docs.heptio.com/content/tutorials/rbac.html

   ```bash
   # # 指定用户名称为 users:jakub, 作者应该使用 jakub 会更好。
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1beta1
   metadata:
     name: jakub-view-all
   subjects:
   - kind: User
     name: users:jakub
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: view
     apiGroup: rbac.authorization.k8s.io
   ```

8. 即使将csr证书删掉以后，kubeconfig文件仍然有效

   ```bash
   ➜  cst kubectl delete csr user-request-jakub
   certificatesigningrequest.certificates.k8s.io "user-request-jakub" deleted
   ➜  cst kubectl --kubeconfig ~/.kube/config-jakub -n kube-system get pods
   NAME                                            READY   STATUS    RESTARTS   AGE
   calico-kube-controllers-5978c5f6b5-qfp9c        1/1     Running   0          4h32m
   ...
   ```

### 为group分配权限

```bash
# 撤回之前的权限
k delete clusterrolebindings.rbac.authorization.k8s.io jakub-view-all
clusterrolebinding.rbac.authorization.k8s.io "jakub-view-all" deleted

# 获取 pod 失败
➜  cst kubectl --kubeconfig ~/.kube/config-jakub get pods
Error from server (Forbidden): pods is forbidden: User "users:jakub" cannot list resource "pods" in API group "" in the namespace "default"

# 修改 ClusterRoleBinding 对象
➜  cst vim cr.yaml
➜  cst k apply -f cr.yaml
clusterrolebinding.rbac.authorization.k8s.io/test-default-view-all configured

# 修改内容
cst cat cr.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: test-default-view-all
subjects:
- kind: Group
  name: "cool-people"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io

# 校验
kubectl --kubeconfig ~/.kube/config-jakub get pods
No resources found in default namespace.
```

## 通过 Token 生成 kubeconfig 文件

1. 获取 token 信息

   ```bash
   # 创建 namespace
   ➜  cst k -n test get sa
   NAME      SECRETS   AGE
   default   1         13s
   
   # 查看 default service account
   ➜  cst k -n test get sa -oyaml | kubectl-neat
   apiVersion: v1
   items:
   - apiVersion: v1
     kind: ServiceAccount
     metadata:
       name: default
       namespace: test
     secrets:
     - name: default-token-rcjlr
   kind: List
   metadata:
     resourceVersion: ""
     selfLink: ""
     
   # 这样也可以。
   # 注意 system:serviceaccount:test service account default 的group
   ➜  cst cat cr.yaml
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1beta1
   metadata:
     name: test-default-view-all
   subjects:
   - kind: User
     name: "system:serviceaccount:test:default"
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: view
     apiGroup: rbac.authorization.k8s.io
   
   # 查看 default service account 对应的 secret 对象
   # 注意 secret 对象的内容都是base64编码的
   k -n test get secret default-token-rcjlr -oyaml | kubectl-neat
   
   # 获取 token 信息
   k -n test get secret default-token-rcjlr -o jsonpath={.data.token} | base64 -d
   ```

2. 查看 token

   ```bash
   # token 是通过 JWT 类型
   ➜  cst k -n test get secret default-token-rcjlr -o jsonpath={.data.token} | base64 -d | jwt decode -
   ```

3. 创建 kubeconfig 文件

   ```bash
   # 创建 cluster
   kubectl --kubeconfig ~/.kube/config-test config set-cluster jakub --insecure-skip-tls-verify=true --server=https://127.0.0.1:61620
   
   # 创建 user
   kubectl --kubeconfig ~/.kube/config-test config set-credentials jakub --token $TOKEN
   
   # 创建 context
   kubectl --kubeconfig ~/.kube/config-test config set-context jakub --cluster=jakub --user=jakub
   
   # 切换 context
   kubectl --kubeconfig ~/.kube/config-test config use-context jakub
   ```

4. 读取pod

   ```bash
   kubectl --kubeconfig ~/.kube/config-test get pods
   
   Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:test:default" cannot list resource "pods" in API group "" in the namespace "default"
   ```

5. 分配权限

   ```bash
   ➜  cst k apply -f cr.yaml
   clusterrolebinding.rbac.authorization.k8s.io/test-default-view-all created
   ➜  cst kubectl --kubeconfig ~/.kube/config-test get pods
   No resources found in default namespace.
   ```

   

## RBAC 最佳实践

![image-20210423155847323](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210423155847323.png)

1. default service account 没有什么权限。前面已经验证。

2. **The most secure option for RBAC is to have your application create a service account for itself with only the permissions it needs.**

3. **省去了创建Role：The next least permissive option is to create a specific service account for an application that has admin access to that application’s namespace.** 

   ```bash
   # 在 varMyNamespace 下，所有的拥有管理员的权限
   kubectl create rolebinding varMyRoleBinding \
     --clusterrole=admin \
     --serviceaccount=varMyNamespace:default \
     --namespace=varMyNamespace
   ```

4. **省去了创建SA：The third option, which is a good quick permissions workaround if less secure, is to grant admin access to the default service account for a particular namespace to that same application namespace.** 

5. The last option to get around the new RBAC permissions without completely turning them off is to give the `kube-system` default service account admin access to the whole cluster.  需要管理员权限的应用，都部署到该 Namespace 下。

   ```bash
   # kube-system namespace 下的Pod拥有集群的管理员权限
   kubectl create clusterrolebinding varMyClusterRoleBinding \
     --clusterrole=cluster-admin \
     --serviceaccount=kube-system:default
   ```

   