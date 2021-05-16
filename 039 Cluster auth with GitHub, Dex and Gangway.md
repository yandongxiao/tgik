# 039: Cluster auth with GitHub, Dex and Gangway

## 什么是 OAUTH 2？

https://datatracker.ietf.org/doc/html/rfc6749

The OAuth 2.0 authorization framework enables a third-party application to obtain limited access to an HTTP service, either on behalf of a resource owner by orchestrating an approval interaction between the resource owner and the HTTP service, or by allowing the third-party application to obtain access on its own behalf. 

1. 基于TLS提出的协议。
2. Web Service 与 Identity Service 之间传递的 token 是 JWT（Java Web Token）

### 交互过程

1. resource owner. An entity capable of granting access to a protected resource. When the resource owner is a **person**, it is referred to as an end-user.
2.  client. An application making protected resource requests on behalf of the resource owner and with its authorization.  
3. resource server. The server hosting the protected resources, capable of accepting and responding to protected resource requests using access tokens
4.  authorization server. 负责签发 Access Token。The server issuing access tokens to the client after successfully authenticating the resource owner and obtaining authorization.

![image-20210516141706787](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210516141706787.png)

站在用户的角度来看：

![image-20210516020752547](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210516020752547.png)

1. 用户打开Application的页面，点击注册；
2. Application 返回一个重定向 redirect。用户随后请求 Authorization Server，请求颁发一个Token。
3. Authorization Server 也会返回一个重定向 redirect。用户随后携带 Access Token 再次访问 Application。
4. Application 访问 Resource Server，获取资源信息，如用户的邮件、姓名、性别等信息。

> Authorization Server 和 Resource Server 可能对外暴露成一个 server。例如，github 既是 Authorization Server，又是 Resource Server

## 什么是OIDC？

全称是：OpenID Connector。

OpenID Connect (OIDC) is an identity layer built on top of the OAuth 2.0 framework. It allows third-party applications to verify the identity of the end-user and to obtain basic user profile information. OIDC uses JSON web tokens (JWTs), which you can obtain using flows conforming to the OAuth 2.0 specifications.  信息来自：https://auth0.com/docs/protocols/openid-connect-protocol

站在用户的角度来看：

![image-20210516021859625](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210516021859625.png)

1. 与 OAuth 2 相比，Access Token 的内容是 JWT + Refresh Token
2. JWT会过期，Refresh Token用于获取新的 Token

## DEX的作用

![image-20210516022211662](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210516022211662.png)

1. 它被称为 OIDC middleware。
2. 对于 gangway 来说，gangwaty 与 Dex 之间使用 OIDC 协议进行通信。Dex 负责签发 JWT 并管理 Token 的有效期。
3. 对 github.com 等 user-management system 来说，Dex 与 它们 之间是通过OAuth 2 协议进行交互。不同的 user-management 的 OAuth 2 的实现是不同的，借助 Dex 的插件体系，可以轻松搞定。

## gangway 的作用

Systems that can already consume OpenID Connect ID Tokens issued by dex include: [Kubernetes](http://kubernetes.io/docs/admin/authentication/#openid-connect-tokens)

1. 如果发送给API Server 的请求中带了JWT Access Token，那么 API Server 会向 Dex 发送请求。
2. 首先，验证 JWT 的有效性
3. 其次，获取 用户的 user, group 信息

> 这些需要在 API Server 的 yaml 文件中进行配置

所以，剩下的一个问题是，如何构造一个 kubeconfig 文件，里面包含了 JWT 的信息？这就是 gangway 要做的事情！！

![image-20210516144916893](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210516144916893.png)

### gangway 根据 OIDC协议获取 Token 的过程

![image-20210516144813434](/Users/yandongxiao/Library/Application Support/typora-user-images/image-20210516144813434.png)

