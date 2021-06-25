# 019: Prometheus as a noob



## 一周大事件

1. 这是2017年的最后一次分享
2. Prometheus 的设计源自于 Borgman
3. Joe 在 kube con 上分享的内容《the road to more usable kubernets》

## Kubernetes 1.9 release

1. Client Go 中，shared informer 支持 namespace 级别的 Watch
2. ETCD 3.0 ==> ETCD 3.1 版本
3. **admission control webhooks. Mutatingadmissionwebhooks.**
4. apps group 下的对象 Deployment, StatefulSet 的版本，有v1beta1 ==> v1。变成了GA版本
5. Aggregate ClusterRole。 You can *aggregate* several ClusterRoles into one combined ClusterRole.
6. Kubeadm 支持 dynamic kubelet configuration。这是一个alpha版本。

## parallel-ssh

要解决的问题：如果 kubelet 的配置需要更新，你需要在每个Node上进行操作。

```shell
brew install pssh

NODES=$(kubectl get nodes -o jsonpath="{range .items[*]}-H ubuntu@{.metadata.name} {end}")

parallel-ssh -i -O StrictHostKeyChecking=no $NODES \
  sudo "sh -c 'sed -e "/cadvisor-port=0/d" -i /etc/systemd/system/kubelet.service.d/10-kubeadm.conf; \
        systemctl daemon-reload;
        systemctl restart kubelet'"
```

## 架构图

![image-20210503100322281](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210503100322281.png)

1. Prometheus Server 有三部分组成：Retrieval, Storage, PromQL。Retrieval 根据 Prometheus Config 从其它服务拉去指标；然后将它存储到时间序列数据库中；对外暴露PromQL，对指标进行运算。
2. Retrieval 部分
   1. 服务也可以将数据推送到 Pushgateway，Prometheus 从 Pushgateway 拉去数据。
   2. Prometheus Server 也可以拉取其它 Prometheus Server 的指标，形成了一个 Prometheus Tree。
3. PromQL 部分
   1. 你可以直接通过PromQL查询指标，也可以通过WEB UI、Grafana 来查看指标。
   2. 通过 PrometheusRules 来定义告警规则，如果触发告警规则，Prometheus 会将告警信息推送到AlertManager。

## 周边工具

1. Prometheus Server 类似于 Nginx，它根据Prometheus Config信息，抓取数据，存储，检查告警规则，告警到AlertManager。
2. Prometheus Operator 则 类似于 Nginx Ingress Controller，它负责收集所有Namespace下的配置片段，形成配置文件，并下发给Prometheus Server。
3. Kube-promethues 则提供了安装 Prometheus Operator，基于 Operator 的工作模式，配置 node-exporter，kube-state-metrics 等ServiceMonitor，最终形成一套安装监控Kubernetes的系统。

## kube-prometheus的工作过程

1. 安装 prometheus-operator。它主要由 deployment, service, rbac, service account 组成。Operator 会安装三个重要的CRD。Prometheus、AlertManager、ServiceMonitor。
   1. Prometheus。一个Prometheus代表了一个Prometheus Server 实例。所以我们可以在Kubernetes中部署多个Prometheus Server实例。比如，你可以为K8S的运维人员安装一个Prometheus实例；为上层应用安装另外一个Prometheus实例。
      1. Prometheus Object 通过 label selector，定位到多个 ServiceMonitor
      2. Prometheus Object 中通过 label selector，定位到多个 PrometheusRule
      3. Prometheus Object 中指明了 Alert Manager 实例
   2. ServiceMonitor 是 Prometheus Config 的配置片段。它指明了去哪个Service，收集对应的指标，这样一个任务。
   3. AlertManager。这是一个服务实例。
2. 安装 ServiceMonitor，包括 node-exporter，kube-state-metrics 。
3. 安装 Grafana。
4. 安装 /manifests/prometheus。它就会创建 Prometheus 对象。
5. 安装 ./manifests/alertmanager。它就会创建 alertManager 对象。

## 对外暴露的服务

Prometheue, AlertManager，Grafana 是通过 Node Type 暴露的三个服务。

### 通过 SSH tunnel 访问 Node

```shell
SSH_KEY=~/.ssh/id_rsa
ssh -i $SSH_KEY -A \
  -L30900:localhost:30900 \
  -L30903:localhost:30903 \
  -L30902:localhost:30902 \
  -o ProxyCommand="ssh -i \"${SSH_KEY}\" ubuntu@x.x.x.x nc %h %p" ubuntu@x.x.x.x
```

## 