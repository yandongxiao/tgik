# 008: Continuing the Controller

## 1. 功能介绍

1. 作者没有准备介绍CRD相关的Controller，也称为Operator
2. 介绍了 Informer 的工作原理
3. 着重介绍了podGetter 和 podLister 的却别。podGetter 是实时从API Server获取最新的Manifest。**注意：podGetter 不仅仅可以用于获取Pod信息，可以对Pod做任何操作。**

## 2. 原先的代码介绍

1. Tgik-controller 在 github.com/jbeda 空间下
2. 所有的 Controller 是否都要放在同一个进程下才是高效的。Joe 的回答是：你可以这样做，但是需要考虑到 Controller 之间会相互影响，比如某个Controller Crash，会导致整个进程重启；破坏了 RBAC 的规则最小化原则。
3. 注意：当你在做cache.WaitForCacheSync时，事件也伴随着发生，而且是Add事件。这是因为：事件类型的判断不是依据API Server来判断的，而是依据Informer cache来判断的。当cache.WaitForCacheSync执行时，Informer cache 中就开始添加元素，相应的 Add 事件就会被触发！

## 3. 解决思路

1. 作者一开始并没有按照 Controller 的思路来写代码。而是写了一个函数，它做了以下事情：
   1. List template namespace 下所有含有某Annotation的Secret列表（src list）
   2. 遍历所有的Namespace
      1. Create/Update all of the secrets in this namespace
      2. Delete secrets that have annotation but are not in our src list

## 4. 开发

1. src list 中的secret对象不可以直接apply到target namespace中。修改如下：

   ```
   // 首先要做deepcopy
   newSecretInf, _ := scheme.Scheme.DeepCopy(secret)
   newSecret := newSecretInf.(*apicorev1.Secret)
   
   // 清除 resourceVersion 和 UID 字段
   newSecret.Namespace = ns
   newSecret.ResourceVersion = ""
   newSecret.UID = ""
   ```

2. 关于错误处理，也有专门的package。

   ```
   # package apierrors "k8s.io/apimachinery/pkg/api/errors"
   #
   _, err := c.secretGetter.Secrets(ns).Create(newSecret)
   if apierrors.IsAlreadyExists(err) {
       log.Printf("Scratch that, updating %v/%v", ns, secret.Name)
       _, err = c.secretGetter.Secrets(ns).Update(newSecret)
   }
   ```

3. 作者的处理方式不够高效，是因为它每次不是Create就是Update。如果namespace中已经存在secret对象，且该secret对象与template namespace中的secret对象完全一样。则没有必要进行Update，因为这会导致no op update事件的产生，加重 API Server 的负担！

