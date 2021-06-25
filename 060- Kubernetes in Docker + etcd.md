# 060: Kubernetes in Docker + etcd

## 关于作者

Kris Nova is a software engineer, alpinist, author, and transgender advocate best known for her work on Kubernetes. She contributes to and maintains multiple parts of Kubernetes and is the creator of kubicorn — a popular Kubernetes infrastructure management tool. In addition, Nova is also the author of “Cloud Native Infrastructure” and an ambassador for the Cloud Native Computing Foundation.

## 一周大事件

1. 2018年12月，在西雅图举行了 KubeCon大会
2. 作者发起了一个 Favorite Talks Of KubeCon Seattle 2018
3. 作者选了三个自己觉得非常好的视频。
4. Faces Of Open Source 是一本书，里面介绍了计算机世界里的牛人们。

![image-20210505005605713](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210505005605713.png)

## Kind 简介

1. 作者非常喜爱这个项目，应该是她的最爱。
2. 她参与的一个项目 test-Infra。每当Kubernetes有新的版本出来时，就会跑 test-Infra 项目中的测试用例。
3. 这部分工作当然可以被 kind 拿来使用。
4. 另一方面，test-Infra 的测试完全可以在 docker 中进行，保证环境干净。

