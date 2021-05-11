# 027: Securing the k8s dashboard and beyond!

## 一周大事件

1. unsplash.com 上面有非常精美的图片
2. 《lessons from the cryptojacking attck at tesla》特斯拉因为把dashboard暴露到公网上，引发的问题。
3. img：在构建容器镜像的过程中，要么使用docker daemon on node，要么使用dind（需要额外的权限），都是有安全问题。img 提供了一种clean way。不需要特权，不需要运行daemon的方式
4. **关于Kubernetes安全相关的话题：《hacking and hardening kubernetest cluster by example》**，来自CNCF大会.
5. 在访问dashboard的时候，你可以通过 browser extension 将 token 放到 Header中，进而可以免登录。
6. dashboard只接受token的校验方式，即使通过kubeconfig进行认证，dashboard也会从中解析出token。

## 关于解决 secret 存在在gitrepo中的问题

1. 使用 sealed-secret。它是对secret对象进行转换，转换成另外一个CRD对象。
   1. 该CRD对象在git repo中保存是安全的。
   2. CRD对象的Controller有私钥，只有在apply到k8s集群时，才会解码，生成Secret对象。
2. 使用 git-crypt ，解决的是相同的问题。

## Token内容

1. secret 中存储的是BASE64编码的 JWT Token
2. 将 JWT Token 放到 JWT 官网上解析，可以获取到明文。

![image-20210504161514350](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210504161514350.png)

## OAUTH2_proxy （3.5k）

A reverse proxy and static file server that provides authentication using Providers (Google, GitHub, and others) to validate accounts by email, domain or group.

![image-20210504162102152](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210504162102152.png)

1. Nginx 的位置应该是 Ingress Controller
2. oauth2_proxy 通过 authentication service provider，确定用户的身份
3. Upstream http server 值得是 dashboard。

## 安装

## 1. 安装 oauth2-proxy

`$ go get github.com/oauth2-proxy/oauth2-proxy` which will put the binary in `$GOROOT/bin`

### 2. 在 OAUTH Provider 上面注册OAUTH应用。[Select a Provider and Register an OAuth Application with a Provider](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider)

![image-20210504163202289](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210504163202289.png)

Authentication Callback URL：这里填写的就是 OAUTH2_proxy 在Internet上暴露的域名。

### 3. `oauth2-proxy` can be configured via [config file](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview/#config-file), [command line options](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview/#command-line-options) or [environment variables](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview/#environment-variables).

```shell
kubectl create secret generic k8s-dashboard-oauth-secrets \
  -o yaml --dry-run \
  -n kube-system \
  --from-literal=client-id=c65d2f658c05aacf2f35 \
  --from-literal=client-secret=8b9ae8d9eee6ce756d4894aca85df19b90d0aa22 \
  --from-literal=cookie=$(python -c 'import os,base64; print base64.urlsafe_b64encode(os.urandom(16))')
```

> client-id, client-secret 都是来自于上一步的结果

#### 4. 部署 OAUTH2_proxy 应用

原文参见这里：https://gist.github.com/jbeda/53a7c6c81359054eacc1608f5211150c

```yaml
---
  apiVersion: v1
  kind: Secret
  metadata:
    name: k8s-dashboard-oauth-secrets
    namespace: kube-system
  type: Opaque
  data:
    # 内容来自上一步
    client-id: YzY1ZDJmNjU4YzA1YWFjZjJmMzU=
    client-secret: OGI5YWU4ZDllZWU2Y2U3NTZkNDg5NGFjYTg1ZGYxOWI5MGQwYWEyMg==
    cookie: eUJJVS1qWEVqcTRUV29rOTVaOHNKdz09
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: k8s-dashboard-oauth-proxy
  name: k8s-dashboard-oauth-proxy
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      run: k8s-dashboard-oauth-proxy
  template:
    metadata:
      labels:
        run: k8s-dashboard-oauth-proxy
    spec:
      containers:
      - args:
        - --cookie-secure=false
        - --provider=github
        # proxy 与 dashboard 之间是http通信，所以 dashboard 的启动参数修改修改。
        - --upstream=http://kubernetes-dashboard.kube-system.svc.cluster.local
        - --http-address=0.0.0.0:8080
        # k8s-dashboard.tgik.io 应该是 proxy 对外暴露的域名
        - --redirect-url=https://k8s-dashboard.tgik.io/oauth2/callback
        - --email-domain=*
        - --github-org=TGIK
        - --pass-basic-auth=false			# 这个Header的值，会使得dashboard它作为Token进行登录
        - --pass-access-token=false		# 这个Header的值，会使得dashboard它作为Token进行登录
        env:
        - name: OAUTH2_PROXY_COOKIE_SECRET
          valueFrom:
            secretKeyRef:
              key: cookie
              name: k8s-dashboard-oauth-secrets
        - name: OAUTH2_PROXY_CLIENT_ID
          valueFrom:
            secretKeyRef:
              key: client-id
              name: k8s-dashboard-oauth-secrets
        - name: OAUTH2_PROXY_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              key: client-secret
              name: k8s-dashboard-oauth-secrets
        image: a5huynh/oauth2_proxy:2.2
        name: oauth-proxy
        ports:
        - containerPort: 8080
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: k8s-dashboard-oauth-proxy
  name: k8s-dashboard-oauth-proxy
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: k8s-dashboard-oauth-proxy
  type: ClusterIP
---
# 这是 certificate manager 定义的 CRD
# 通过一个 secret 对象，生成对应 Certificate 对象
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: k8s-dashboard-tls
  namespace: kube-system
spec:
  # 输入是一个 secret 对象
  secretName: k8s-dashboard-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName:  k8s-dashboard.tgik.io
  dnsNames:
  -  k8s-dashboard.tgik.io
  acme:
    config:
    - http01: {}
      domains:
      -  k8s-dashboard.tgik.io
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: contour
    # ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: k8s-dashboard.tgik.io
    http:
      paths:
      - backend:
          serviceName: k8s-dashboard-oauth-proxy
          servicePort: 8080
        path: /
  tls:
  - hosts:
    - k8s-dashboard.tgik.io
    secretName: k8s-dashboard-tls
```

1. Joe 完成了认证这一步，页面会跳转到Dashboard的登录页面。
2. 在Dashboard登录页面，你仍然需要提供对应的Token。
3. proxy 在向 upstream 转发请求时，会带上一个X-Forward-Access-Token。用户可以添加一个WebHook，向 github.com 发送请求，验证Token的有效性。继而确认用户的身份。![image-20210504172422667](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210504172422667.png)

4. Dashboard 和 proxy 的关系是一对多，所有想暴露的服务都可以通过proxy这一关的话，服务的安全性就比较高。这时服务和proxy之间就是多对一的关系。

## 问题

![image-20210504165357206](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210504165357206.png)

