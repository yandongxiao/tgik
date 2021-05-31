# 101: Grokking Kubernetes - API Server: Part 2/X

## 简介

1. 28:00 开始介绍 API Server
2. kubectl auth can-i --list 展示可以操作的资源
3. 通过 kubectl 的 --token 参数，可以覆盖当前kubeconfig file 中的 token。这对于遍历一个 namespace 下的token，结合 kubectl auth，总结它们的权限，是很有帮助的。

kube-apiserver



-  

  

   

  docs

  -  , jwts, tokens
  -  Kubernetes components
  -  People and bots

-  

  How the apiserver handles authorization

   

  docs

  - [x]

-  

  Admission Control

   

  docs

  -  builtins
  -  PSP

-  

  Exploring the API.

  -  kubectl explain
  -  openapi stuffs
  -  deprecation [docs](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
  -  [api reference docs](https://kubernetes.io/docs/reference/using-api/api-concepts)
  -  validating and mutating webhooks [docs](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
  -  static pods

-  

  theory of operation

  -  access patterns.