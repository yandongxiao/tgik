# 011: Upgrading to 1.8 with kubeadm

## 介绍升级前的安装事项

这是不完全的事项，详情参见https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.8.md#before-upgrading:

- The kubelet now fails if swap is enabled on a node. To override the default and run with /proc/swaps on, set `--fail-swap-on=false`. The experimental flag `--experimental-fail-swap-on` is deprecated in this release, and will be removed in a future release. 作者认为在生产环境开启swap是不合适的，因为它会使得性能表现得不够平稳。swap就是将最不常用应用的内存置换到磁盘上，以作他用。
- The `autoscaling/v2alpha1` API is now at `autoscaling/v2beta1`. However, the form of the API remains unchanged. Migrate the `HorizontalPodAutoscaler` resources to `autoscaling/v2beta1` to persist the `HorizontalPodAutoscaler` changes introduced in `autoscaling/v2alpha1`. The Horizontal Pod Autoscaler changes include support for status conditions, and autoscaling on memory and custom metrics.  作者介绍了alpha版本的最大问题：除了API接口会有改动，不向后兼容，还存在**升级问题**。
- The metrics APIs, `custom-metrics.metrics.k8s.io` and `metrics`, were moved from `v1alpha1` to `v1beta1`, and renamed to `custom.metrics.k8s.io` and `metrics.k8s.io`, respectively. If you have deployed a custom metrics adapter, ensure that it supports the new API version. If you have deployed Heapster in aggregated API server mode, upgrade Heapster to support the latest API version. 这个是一个API改动
- StatefulSet: The deprecated `pod.alpha.kubernetes.io/initialized` annotation for interrupting the StatefulSet Pod management is now ignored. StatefulSets with this annotation set to `true` or with no value will behave just as they did in previous versions. Dormant StatefulSets with the annotation set to `false` will become active after upgrading. Annotation 一般不作为API的一部分，但是用的认多了，对Annotation的修改也需要放到Change Log中。
- The CronJob object is now enabled by default at `v1beta1`. CronJob `v2alpha1` is still available, but it must be explicitly enabled. We recommend that you move any current CronJob objects to `batch/v1beta1.CronJob`. Be aware that if you specify the deprecated version, you may encounter Resource Not Found errors. These errors occur because the new controllers look for the new version during a rolling update. CronJob变成了beta版本。
- The `system:node` role is no longer automatically granted to the `system:nodes` group in new clusters. The role gives broad read access to resources, including secrets and configmaps. Use the `Node` authorization mode to authorize the nodes in new clusters. To continue providing the `system:node` role to the members of the `system:nodes` group, create an installation-specific `ClusterRoleBinding` in the installation. ([#49638](https://github.com/kubernetes/kubernetes/pull/49638)) Node Controller的授权不是走的RBAC，而是node authorizor



