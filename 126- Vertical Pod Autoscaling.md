[TOC]

# 126: Vertical Pod Autoscaling （4.5k star）

## 三种 autoscaling 的方式

1. HPA（水平扩展 Pod）。解决的问题是：当不可预知的事件发生时，当前服务的副本数不足以撑起整个负载。
2. Cluster Node Autoscaling。实现这种动态扩缩容的能力，难点在于 Kubernetes 目前尚未提供稳定的 Cluster API，比如，不存在 Add Node To Cluster，这样的API。
3. VPA。解决的问题：如何智能地设置 CPU 和 MEM 的 Request 和 Limit 值。谷歌有篇论文：《Workload Autoscaling at Google》。Kubernetes 与 Borg 的另外一个区别：在Kubernetes中，对 Request 和 Limit 值的调整，会导致 Pod 的重启。

## VPA On Java 

1. 在 Kubernetes 中，对 Java 设置 Request 和 Limit，实际是对 JVM 的堆大小进行限制。
2. 在 Java 应用中，还可能会使用 堆外内存，比如 JNI 使用的内存。
3. 从 kubelete 角度看，java 应用可以使用的内存（Limit值） == 堆内存 和 堆外内存。
4. 从 JVM 角度看，Java 应用可以使用的内存（Limit值） == 堆内存，堆外内存不受限制。
5. 难点，你不能以 GC 作为衡量 Request和 Limit 的标准

## Metric Server

1. 这是安装 VPA 的前提条件
2. 你可以在 kind 上，运行 metric server。前提是需要开启 insecure 模式。
3. 你可以使用 kuard 来模拟CPU和MEM的负载。
