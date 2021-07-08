# 009: Finishing the Controller

## 1. 事件介绍

1. heptio公司完成了B轮融资
2. envoy 被纳入CNCF 基金会

## 2. 代码回顾

1. 作者采用了 one-shot 模式来实现代码。
2. 作者在事件处理时直接执行 one-shot 逻辑。
3. 在做cache.WaitForCacheSync时，事件也伴随着发生，而且是Add事件。
   1. 为什么两者没有先后顺序？事件的处理是由另外一个协程来处理的。
   2. 为什么会产生Add事件？事件类型的判断不是依据API Server来判断的，而是依据Informer cache来判断的。当cache.WaitForCacheSync执行时，Informer cache 中就开始添加元素，相应的 Add 事件就会被触发！
4. 根据2和3可知，事件触发和cache.WaitForCacheSync逻辑是并行的。如果直接在事件处理函数中执行one-short，会导致出错。需要引入queue。

## 3. 代码实现

```go
	log.Print("waiting for cache sync")
	if !cache.WaitForCacheSync(
		stop,
		c.secretListerSynced,
		c.namespaceListerSynced) {
		log.Print("timed out waiting for cache sync")
		return
	}
	log.Print("caches are synced")

	go func() {
		// runWorker will loop until "something bad" happens. wait.Until will
		// then rekick the worker after one second.
		wait.Until(c.runWorker, time.Second, stop)
		// tell the WaitGroup this worker is done
		wg.Done()
	}()

```

1. 业务逻辑的代码的处理 和 `WaitForCacheSync` 放在了一个协程，明确了先后顺序
2. 作者在queue里面，每次都放置的是同一个字符串。这与 one-short 模式是匹配的。
3. `queue workqueue.RateLimitingInterface`的回退策略对API Server 也是一种保护

## 问题

新增一个namespace时会触发one-short逻辑的执行。它是否会引起Update风暴？**不会。而且Update操作返回成功。**

```go
// 1. Create/Update all of the secrets in this namespace
	for _, secret := range secrets {
		newSecretInf, _ := scheme.Scheme.DeepCopy(secret)
		newSecret := newSecretInf.(*apicorev1.Secret)
		newSecret.Namespace = ns
		newSecret.ResourceVersion = ""
		newSecret.UID = ""

		log.Printf("Creating %v/%v", ns, secret.Name)
		_, err := c.secretGetter.Secrets(ns).Create(newSecret)
		if apierrors.IsAlreadyExists(err) {
			log.Printf("Scratch that, updating %v/%v", ns, secret.Name)
			_, err = c.secretGetter.Secrets(ns).Update(newSecret)
		}
		if err != nil {
			log.Printf("Error adding secret %v/%v: %v", ns, secret.Name, err)
		}
	}
```

1. user3 namespace 的 annotation 更新，触发了下面逻辑：

![image-20210425231446718](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210425231446718.png)

2. default, user1, user2 namespace下，并没有发生Update事件的更新。
3. User3 namespace 下面添加了ss secret，无法解释随后的Update事件。
4. 根据代码可以解释为：
   1. 发生Create操作时，newSecret的某些字段被K8S被忽略了。
   2. 发生Update操作时，这些字段没有被忽略。

## 尝试找到的答案

1. 我们手动模拟上面的场景。一次 create，多次 apply。它们的都是同一个yaml文件
2. 结果：产生一次Add事件，两次apply事件。
3. 结论是：在最新的K8S中，作者Update相同的内容时，可能会产生不止一次Update事件。

### Add事件结果

```yaml
apiVersion: v1
data:
  key1: c3VwZXJzZWNyZXQ=
kind: Secret
metadata:
  creationTimestamp: "2021-04-25T15:27:01Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:key1: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-04-25T15:22:51Z"
  name: foo
  namespace: xxx
  resourceVersion: "267581"
  selfLink: /api/v1/namespaces/xxx/secrets/foo
  uid: 41549914-22f3-4d31-ba3b-38b889bc7626
type: Opaque
➜  ~
```

### Update-1

多出了Annotation字段，resourceVersion也不一样了。

```yaml
apiVersion: v1
data:
  key1: c3VwZXJzZWNyZXQ=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"key1":"c3VwZXJzZWNyZXQ="},"kind":"Secret","metadata":{"annotations":{},"creationTimestamp":"2021-04-25T15:22:51Z","managedFields":[{"apiVersion":"v1","fieldsType":"FieldsV1","fieldsV1":{"f:data":{".":{},"f:key1":{}},"f:type":{}},"manager":"kubectl-create","operation":"Update","time":"2021-04-25T15:22:51Z"}],"name":"foo","namespace":"xxx","selfLink":"/api/v1/namespaces/kube-public/secrets/foo"},"type":"Opaque"}
  creationTimestamp: "2021-04-25T15:27:01Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:key1: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-04-25T15:22:51Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2021-04-25T15:27:18Z"
  name: foo
  namespace: xxx
  resourceVersion: "267611"
  selfLink: /api/v1/namespaces/xxx/secrets/foo
  uid: 41549914-22f3-4d31-ba3b-38b889bc7626
type: Opaque
```

### Update-2

与Update-1相比，managedFields添加了一个元素。

```yaml
apiVersion: v1
data:
  key1: c3VwZXJzZWNyZXQ=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"key1":"c3VwZXJzZWNyZXQ="},"kind":"Secret","metadata":{"annotations":{},"creationTimestamp":"2021-04-25T15:22:51Z","managedFields":[{"apiVersion":"v1","fieldsType":"FieldsV1","fieldsV1":{"f:data":{".":{},"f:key1":{}},"f:type":{}},"manager":"kubectl-create","operation":"Update","time":"2021-04-25T15:22:51Z"}],"name":"foo","namespace":"xxx","selfLink":"/api/v1/namespaces/kube-public/secrets/foo"},"type":"Opaque"}
  creationTimestamp: "2021-04-25T15:27:01Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:key1: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-04-25T15:22:51Z"
  name: foo
  namespace: xxx
  resourceVersion: "267624"
  selfLink: /api/v1/namespaces/xxx/secrets/foo
  uid: 41549914-22f3-4d31-ba3b-38b889bc7626
type: Opaque
```

