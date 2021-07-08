[TOC]

# 142: Carvel （kbld）

carvel 有很多精巧的小工具，比如：ytt（yaml templating toll），kapp，kbld，imgpkg。

## kbld

它是构建过程中，可能要用到的工具，但是与 docker， buildkit， kaniko， jib（for java），gitlab-runner in Docker 并不冲突。它是一个命令行工具：yaml in, yaml out，

1. kbld behind the scenes uses existing mature products to build images (such as Docker and Buildpack's pack) and automatically updates your configuration with the newly built digest references。所以，kbld 是更新 Deployment 中 image 参数的一种方式。 
2. **YAML in, YAML out。**kbld works with any YAML configuration (e.g. Kubernetes resources), hence can work with wide variety of deployment tools。
3. 该工具的常见用途，使用 digest reference 唯一地标识容器镜像；添加源代码的信息；**容器镜像迁移**。

### 容器镜像迁移

1. 容器镜像迁移是指：从 docker registry A 迁移到 docker registry B。kbld 支持两种迁移方式：relocation 和 packaging 方式。
2. `kbld relocate` (available v0.23.0+) allows to efficiently copy images between registries as long as running `relocate` command has connectivity to both registries.
3. `kbld package` and `kbld unpackage` allows to export images into a single tarball, and later import them from given tarball into a different (or same) registry. This approach does *not* require connectivity to source registry during the `pkg unpackage` time.

### 例子

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  labels:
    app: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: app1
        ports:
        - containerPort: 80
---
apiVersion: kbld.k14s.io/v1alpha1
kind: Config
sources:
- image: app1 	#（与Deployment中的image字段的值，保持相等）
  path: . # <-- where to find app1 source（从哪里找到源代码，执行docker build）
destinations:	# 注意 Destination 是必须的
- image: app1
  # 构建出来的容器镜像，需要推送到哪个仓库？
  newImage: docker.io/hk/simple-app # <-- where to push app1 image
```

2. output

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  labels:
    app: app1
  annotations:
    # informational metadata about how image was built
    kbld.k14s.io/images: |
      - Metas:
        - Path: /Users/pivotal/workspace/simple-app
          Type: local
        - Dirty: false
          RemoteURL: git@github.com:k14s/super-secret-simple-app
          SHA: e877718521f7ccea0ab0844db0f86fe123a8d8ef
          Type: git
        URL: index.docker.io/hk/simple-app@sha256:e932e46fd...      
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        # 这是推送到镜像仓库的地址
        image: index.docker.io/hk/simple-app@sha256:e932e46fd... # <-- built and pushed image
        ports:
        - containerPort: 80
```

1. 在镜像仓库中，除了常规的 tag（v1.0.0），在 docker hub 上还会有 `kbld-rand tag`。这是为什么？image build in Docker daemon are not assigned digests
2. Annotation 中还会包含源代码相关的信息。
3. `index.docker.io/hk/simple-app@sha256:e932e46fd...` 这种方式称为 digest references，常见方式称为 tag references。后者的缺陷是，tag一致会导致k8s中运行的容器镜像内容不一致。比如 latest。 

## imgpkg

1. 主要功能是将 yaml 文件（实际上可以是任何东东）打包成 image。`imgpkg` is a tool that allows users to store a set of arbitrary files as an OCI image. **One of the driving use cases is to store Kubernetes configuration (plain YAML, ytt templates, Helm templates, etc.) in OCI registry as an image。**

2. imgpkg 不仅仅是将 yaml 文件打包并上传。它还实现了 yaml 文件**依赖**容器镜像的功能。这样做的好处是，确保依赖的容器镜像不会被误删除。A bundle is an OCI image that holds 0+ arbitrary files *and* 0+ references to dependent OCI images (which *may* also be [bundles](https://carvel.dev/imgpkg/docs/latest/resources/#nested-bundle)). By tracking dependent images, imgpkg can copy bundles across registries。

   ![image-20210606194552349](/Users/dxyan06/Library/Application Support/typora-user-images/image-20210606194552349.png)

### 例子

1. Creating the bundle

```bash
examples/basic-step-2
├── .imgpkg
│   ├── bundle.yml
│   └── images.yml
└── config.yml
```

images 文件可以自动生成。

```
kbld -f config.yml --imgpkg-lock-output .imgpkg/images.yml
```

2. Pushing the bundle to a registry

```bash
imgpkg push -b index.docker.io/user1/simple-app-bundle:v1.0.0 -f examples/basic-step-2
```

3. Pulling the bundle to registry

```bash
$ imgpkg pull -b index.docker.io/user1/simple-app-bundle:v1.0.0 -o  /tmp/simple-app-bundle
$ kbld -f ./config.yml -f .imgpkg/images.yml | kubectl apply -f-
```

