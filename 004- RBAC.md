# 004: RBAC

## ### kubeconfig

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

1. 客户端与API Server发送HTTPS请求，建立TLS链接
2. API Server 将自己的证书发送给客户端
3. 客户端将证书发送给CA（API Server），请求CA验证证书的合法性。
4. 验证通过后，用户携带自己的公钥发送请求到服务端（API Server）
5. API Server 根据认证插件，识别出用户的身份。
6. API Server 根据授权插件，识别出用户是否有权限发起该请求。



## 认证

https://kubernetes.io/docs/reference/access-authn-authz/authentication

### X509 Client Certs

1. pem: 私钥信息
2. csr: certificate signing request
3. crt: 由CA签发的证书

1. 信息来源：https://www.linkedin.com/pulse/adding-users-quick-start-kubernetes-aws-jakub-scholz

1. 生成私钥

   ```
   openssl genrsa -out jakub.pem 2048
   ```

2. 生成CSR（certificate signing request）

```
openssl req -new -key jakub.pem -out jakub.csr -subj "/CN=users:jakub/O=cool-people"
```

3. 准备CSR请求。

```
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

注意需要Base64编码。

```
# pbcopy 复制到剪切板
cat jakub.csr | base64 | tr -d '\n' | pbcopy
```

4. 签发证书

   ```
   kubectl certificate approve user-request-jakub
   ```

5. 获取证书

```
kubectl get csr user-request-jakub -o jsonpath='{.status.certificate}' | base64 -d > jakub.crt
```

##  授权（RBAC）

1. 官网：https://kubernetes.io/docs/reference/access-authn-authz/authorization/
2. htptio的教程：http://docs.heptio.com/content/tutorials/rbac.html

## RBAC 最佳实践

![image-20210423155847323](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210423155847323.png)

1. default service account 没有什么权限。
2. **The most secure option for RBAC is to have your application create a service account for itself with only the permissions it needs.**
3. **省去了创建Role：The next least permissive option is to create a specific service account for an application that has admin access to that application’s namespace.** 

4. **省去了创建SA：The third option, which is a good quick permissions workaround if less secure, is to grant admin access to the default service account for a particular namespace to that same application namespace.** 

```
# 在 varMyNamespace 下，所有的拥有管理员的权限
kubectl create rolebinding varMyRoleBinding \
  --clusterrole=admin \
  --serviceaccount=varMyNamespace:default \
  --namespace=varMyNamespace
```

5. The last option to get around the new RBAC permissions without completely turning them off is to give the `kube-system` default service account admin access to the whole cluster. 

```
# kube-system namespace 下的Pod拥有集群的管理员权限
kubectl create clusterrolebinding varMyClusterRoleBinding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:default
```