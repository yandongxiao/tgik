## [00:15:11](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=911s) - Hierarchical Namespace Overview 

1. 随着事件的推移，Team 和 Namespace 之间的关系可能会更改。由一对一的关系，转变为一对多的关系。
2. 你希望在创建 Namespace Y 的时候，可以从Namespace X 继承一些资源，比如 ResourceQuota、NetworkPolicy、Role、RoleBinding 等。
3. 随着事件的推移，你希望这个“继承关系”依然有效。当然，你还希望自定义自己的一部分行为。

### 常见的实现方式

![image-20210615234756116](https:\/\/raw.githubusercontent.com\/yandongxiao\/typera\/main/img/image-20210615234756116.png)

1. 实现一个 NamespaceCreator，它拥有自己的CRD对象
2. NamespaceCreator 监听该对象。
3. 一旦 CRD 对象被创建，NamespaceCreator 将它转移成相应的Manifest。
4. Manifest被创建后，无法控制它们直接被原地修改。NamespaceCreator 更像是一次性操作。

## [00:30:05](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=1805s) - Installing HNC and the Plugin 

1. 由两个组件组成：一个是 HNC Controller，另一个是 kubectl plugin。
2. Manifest 组成：Namespace，CRD、Role、ClusterRole、RoleBinding、MutatingWebhookConfiguration、Deployment。
3. 该项目是由 kubebuilder 作为框架来构建的。

## [00:40:53](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=2453s) Creating Namespaces



## [00:54:38](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=3278s) - Synchronizing Objects

 

## [01:15:32](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=4532s) - Experiments with NS Ownership and more! 



## [01:31:53](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=5513s) - Advanced Resource Quota Propagation Ideas 



## [01:37:05](https://www.youtube.com/watch?v=zAtaJ0x-ZwY&list=PL7bmigfV0EqQzxcNpmcdTJ9eFRPBe-iZa&index=25&t=5825s) - Wrap Up