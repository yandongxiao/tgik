# 030: Exploring Skaffold

## skaffold 简介（11k v1.23）

1. Google Gloud Platform 开源的一个面向开发者的工具。
2. 它通过监控文件的修改，自动地进行 build，push，deploy 操作
3. Joe 建议使用 goreleaser 来生成release相关的内容。可以参考 ksonnet 是如何使用它的。
4. 如果你在本地构建和上传容器镜像，那么构建过程中的依赖（go mod vendor 相关）自然能缓存。如果使用 GCP Container Builder，如何缓存这些内容？
5. 如果skaffold的构建过程慢，也是你优化Dockerfile的好时机。
6. 你需要创建 skaffold.yaml 文件，描构建和部署过程。
   1. 支持 kubernetes manifests
   2. 支持 Helm
   3. 支持多应用
7. 与 Goland IDE 是否进行了很好的整合？

## feature

- Blazing fast local development
  - **optimized source-to-deploy** - Skaffold detects changes in your source code and handles the pipeline to **build**, **push**, and **deploy** your application automatically with **policy based image tagging**
  - **continuous feedback** - Skaffold automatically aggregates logs from deployed resources and forwards container ports to your local machine
- Project portability
  - **share with other developers** - Skaffold is the easiest way to **share your project** with the world: `git clone` and `skaffold run`
  - **context aware** - use Skaffold profiles, user level config, environment variables and flags to describe differences in environments
  - **CI/CD building blocks** - use `skaffold run` end-to-end, or use individual Skaffold phases to build up your CI/CD pipeline. `skaffold render` outputs hydrated Kubernetes manifests that can be used in GitOps workflows.
- Pluggable, declarative configuration for your project
  - **skaffold init** - Skaffold discovers your files and generates its own config file
  - **multi-component apps** - Skaffold supports applications consisting of multiple components
  - **bring your own tools** - **Skaffold has a pluggable architecture to integrate with any build or deploy tool**
- Lightweight
  - **client-side only** - Skaffold has no cluster-side component, so there is no overhead or maintenance burden
  - **minimal pipeline** - Skaffold provides an opinionated, minimal pipeline to keep things simple

## 其它问题

### Image Tag是如何生成的？

1. image tag 的生成策略是可以在 skafflold.yaml 文件中指定的
2. 默认的 Image Tag 是 images with their sha256 digest。

### 如果希望使用宿主机的Docker Daemon，一定需要docker login吗？

不一定。使用 docker credential helper 解决。相当用 kubectl context 功能。

### 优化 Kubernetes logs 的视觉体验

Kubernetes kail

### headless的缺点

1. 当通过service name来访问 headless service 时，DNS解析出一个静态的IP列表。
2. http.GET（"service.name"） 会将请求总是发送到某个Pod上。负载均衡功能消失。
3. ClusterIP的访问方式，一定是负载均衡的吗？go http client 默认使用 keep alive的连接方式，http client 发送请求时会优先复用连接。



### 