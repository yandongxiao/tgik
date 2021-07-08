[TOC]

# 039: Cluster auth with GitHub, Dex and Gangway

## 什么是 OAUTH 2？

### 这个协议要解决的问题是什么？

假如，你有 Github 账号，也在 Github 上创建了很多项目。现在，有一个应用（Application）提供某种服务，它可能需要读取你在 Github 上的账号信息 & 项目数据等。如何解决？方法一：应用获取你的Github的用户名和密码。显然这种方式有以下问题：

1. 更新 Github 密码将变得困难。
2. 你无法控制 Application 的操作权限，万一它把你的项目给删除掉呢？

### OAUTH 2 简介

官网信息：https://datatracker.ietf.org/doc/html/rfc6749。

摘要：The OAuth 2.0 authorization framework **enables a third-party application to obtain limited access to an HTTP service**, either on behalf of a resource owner by orchestrating（第三方无法返回一个重定向，用户通过重定向链接，打开Github的页面。这个交互页面上，会提示用户要授权哪些权限） an approval **interaction** between the resource owner and the HTTP service（也称为 Resource Server，比如Github）, or by allowing the third-party application to obtain access on its own behalf（暂不考虑这种方式）. 

### OAuth 1 VS OAuth 2

1. OAuth 1 是基于 HTTP 协议提出的，交互过程复杂。
2. OAuth 2 是基于 TLS 协议提出的，交互过程简单。

### 交互过程

#### 整体交互过程

1. **Resource Owner.** An entity capable of granting access to a protected resource. When the resource owner is a **person**, it is referred to as an end-user.
2. **Client**. An application making protected resource requests on behalf of the resource owner and with its authorization.  
3. **Resource Server.** The server hosting the protected resources, capable of accepting and responding to protected resource requests using **access tokens**
4. **Authorization Server.** 负责签发 Access Token。The server issuing access tokens to the client after successfully authenticating the resource owner and obtaining authorization.

![image-20210516141706787](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210516141706787.png)

> Authorization Server 和 Resource Server 可能对外暴露成一个 server。例如，Github 既是 Authorization Server，又是 Resource Server

#### Authorization Grant 的具体过程

站在用户的角度来看：

![image-20210516020752547](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210516020752547.png)

1. 用户通过浏览器打开 Application 的页面，点击注册。Application 返回一个重定向 Redirect 。
2. 浏览器随后请求 Gmail Server。Gmail Server 返回的页面会提示用户：你想将X、Y、Z的访问权限，赋予给 Application？X、Y、Z 构成的权限列表，被称为 scopes。
3. 用户点击Yes，Gmail Server 则会返回一个重定向 Redirect，同时返回一个 A Single Use Limited Token。
4. 浏览器携带该 Token，访问 Application。
5. Application 携带 Access Token 再次访问 Application。获取一个会过期的 Refresh Token。

接下来，Application 就可以使用 Refresh Token，直接与 Gmail Server通信，获取资源。

> 用户可以登录 Gmail Server，查看赋予给 Application 的权限列表。
> 同时，随时可以撤销 Application 的权限。

## 什么是 OIDC 协议？

全称是：OpenID Connector。详细信息：https://auth0.com/docs/protocols/openid-connect-protocol。

### 这个协议要解决的问题是什么？

OAuth 2 协议是一个授权协议，用户拿到 Access Token 后，就可以获取权限内的资源信息。大多数情况下，你可能只希望Application获取到你的身份信息，比如，我的昵称，邮箱地址等基本的身份信息。

OpenID Connect (OIDC) is an identity layer built **on top of the OAuth 2.0 framework**（协议流程与OAuth2一致）. It allows third-party applications to verify the identity of the end-user and **to obtain basic user profile information.** OIDC uses JSON web tokens (JWTs), which you can obtain using flows conforming to the OAuth 2.0 specifications.

### OIDC VS OAuth 2

![image-20210516021859625](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210516021859625.png)

1. 与 OAuth 2 的交互过程完全一样。不同的是 OIDC Provider（Github）返回的 Access Token 包括两部分：JWT + Refresh Token。
2. JWT 中保存了用户的身份信息。
3. JWT会过期，Application 通过调用 Refresh Token 获取新的 Token。

## 什么是 DEX？

### 这个协议要解决的问题是什么？

1. DEX 被称为 OIDC Middleware。它可以处理与Github、Gmail通信（OAuth 2 协议）；也支持 LDAP协议。
2. 对 github.com 等 user-management system 来说，Dex 与 它们 之间是通过OAuth 2 协议进行交互。不同的 user-management 的 OAuth 2 的实现是不同的，借助 Dex 的插件体系，可以轻松搞定。
3. 对于 gangway 来说，gangwaty 与 Dex 之间使用 OIDC 协议进行通信。**Dex 负责签发 JWT 并管理 Token 的有效期。**

### 交互过程

![image-20210516022211662](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210516022211662.png)

1. 用户访问 ganway，ganway 返回重定向，到Dex。
2. 用户浏览器访问 Dex，Dex 返回重定向，到 Github。
3. 用户选择要权限列表（可能有这个页面），成功后，一步步重定向到 ganway。
4. Ganway 通过 Single Use Token, 访问 Dex，获取 JWT + Refresh Token。**Dex 负责签发 JWT 并管理 Token 的有效期。**
5. ganway 构建出来 kubeconfig 文件（Token 应该就是JWT），调用 kubectl 命令。
6. kubectl 访问 API Server，API Server 访问 Dex 获取 Token 的有效性（Kubernetes supports [OpenID Connect Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) as a way to identify users who access the cluster ，如果发送给 API Server 的请求中带了JWT Access Token，那么 API Server 会向 Dex 发送请求。首先，验证 JWT 的有效性；其次，获取 用户的 user, group 信息。Dex 的 URL 需要在 API Server 的 yaml 文件中进行配置）。

> 对于 API Server 来说，完成身份认证即可。因为用户的权限信息，跟 RBAC 规则有关系。
>
> Kubernetes 支持 OIDC：https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens。

### Get Started

```bash
# 构建 dex binary
make build
# 构建 example-app
make examples

# 修改 examples/config-dev.yaml
connectors:
- type: github
  # Required field for connector id.
  id: github
  # Required field for connector name.
  name: GitHub
  config:
    # Credentials can be string literals or pulled from the environment.
    clientID: a0aa9b25d9ff00af1111
    clientSecret: a78724dc2c2fc14880f69af6214a0605d92c1111
    redirectURI: http://127.0.0.1:5556/dex/callback

# 启动 dex
./bin/dex serve examples/config-dev.yaml

# 启动 example-app
./bin/example-app
2021/07/07 05:55:20 listening on http://127.0.0.1:5555
```

1. Navigate to the example app at http://localhost:5555/ in your browser.

   ![image-20210707180422291](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210707180422291.png)
   1. 你可以填充这些选项，作为 Request Body，访问 Dex Server。也可以直接点击【Login】

2. Hit “login” on the example app to be redirected to dex.

   1. “Login with Example” to use mocked user data.

   2. “Login with Email” to fill the form with static user credentials `admin@example.com` and `password`.

      ![image-20210707191706842](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210707191706842.png)

3. Approve the example app’s request. 如果你在该浏览器中没有登录过 Github，可能需要输入自己的用户名好密码。后续 Login 不会有该操作。

<img src="https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210707181149513.png" alt="image-20210707181149513" style="zoom:50%;" />

4. See the resulting token the example app claims from dex.

![image-20210707192108688](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210707192108688.png)

1. 参见：https://www.c-sharpcorner.com/article/accesstoken-vs-id-token-vs-refresh-token-what-whywhen 文章，解释 ID Token & Access Token & Refresh Token。
2. Access token used in token-based authentication to gain access to resources by using them as bearer tokens。
   1. Access tokens are used as bearer tokens. A bearer token means that the bearer (who holds the access token) can access authorized resources without further identification.
   2. These tokens usually have a short lifespan for security purpose. 
   3. The application should treat access tokens as opaque strings since they are meant for APIs.
   4. 为什么要返回 Access Token？为了与 OAuth 2 协议的兼容性。
3. ID token carries identity information encoded in the token itself, which must be a JWT. It must not contain any authorization information, or any audience information — it is merely an identifier for the user。JWT中仅仅包含了用户的一些身份信息。
4. ID Tokens are [JSON Web Tokens ](https://jwt.io/)(JWTs) signed by Dex and returned as part of the OAuth2 response that attest to the end user’s identity。Systems that can already consume OpenID Connect ID Tokens issued by Dex include: Kubernetes & AWS STS。
5. Refresh tokens are credentials used to obtain access tokens. Refresh tokens are issued to the client by the authorization server and are used to obtain a new id token when the current id token becomes invalid or expires. 如果后端连接是 Github，该字段可能不存在。

## [gangway](https://github.com/heptiolabs/gangway)

### ganway 要解决的问题

要解决的问题。动态构造一个 kubeconfig 文件，里面包含了 JWT 的信息，这就是 gangway 要做的事情。下面的提示信息：创建 kubectl 命令；创建 kubeconfig.yaml 文件。

![image-20210516144916893](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210516144916893.png)

### user、gangway、OIDC 之间的交互过程

![image-20210516144813434](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210516144813434.png)
