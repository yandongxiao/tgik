# 007: Building a Controller

### controller 中的 helloworld

1. https://github.com/jbeda/tgik-controller
2. https://github.com/kubernetes/sample-controller

### controller的基础知识

1. k8s.io/client-go 可以被认为是访问K8S的SDK。但是 client-go 不仅仅可以做Get、List、POST等操作，它更重要的是，方便开发人员编写Controller。
2. podGetter 会从 API Server 获取对象。
3. podLister 会从 client-go 获取对象

### 任务

作者要实现这样一个Controller：当 namespace A 中创建了指定类型的Service对象，则在其它namespace中创建副本。所以他有如下要做的事情：

1. controller 需要有podGetter, podLister, namespaceGetter, namespaceLister
2. 通过 Informer 对 service 进行 List And Watch
3. 通过 Informer 对 namespace 进行 List And Watch

> List 操作会从cache中获取数据，所以 Controller Run 起来以后，第一次List的操作**可能**会被阻塞一段时间。TODO：需要确认下，cache.WaitForCacheSync 是否是同步等待缓存的初始化。

这里的难点在于：

1. 在 stop-start controller 期间，namespace A 中的Service对象被删除。如何在其它namespace中进行同步

### 关于删除

关于删除，需要特殊处理`DeletedFinalStateUnknown`的情况。

```golang
func (c *TGIKController) onDelete(obj interface{}) {
	key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
	if err != nil {
		runtime.HandleError(err)
	}
	log.Printf("onDelete: %v", key)
	c.handleSecretChange(obj)
}

func (c *TGIKController) handleSecretChange(obj interface{}) {
	secret, ok := obj.(*apicorev1.Secret)
	if !ok {
		// TODO: this is probably a `DeletedFinalStateUnknown`.  Figure out what
		// to do.
		return
	}
	...
}
```