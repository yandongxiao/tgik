# 020: Argo Workflow System【知乎】

## 什么是 Workflow System？

1. **Workflow System 有一个智能地调度模块。** 你有一些列的 tasks 要完成。某些 tasks 之间需要串行处理，因为下一个 task 的输入依赖上一个 task 的输出。某些 tasks 之间可以并行处理，加快速度。无论是 argo workflow 还是 tekton 都采用了有向无环图的方式，自动识别 task 之间的依赖关系。
2. **Workflow System 有一个workflow进度追踪模块。**如果某个 work 失败了，你可以快速定位到，可以进行问题排查，可以进行重试。最好有一个UI界面，可以查看 workflow 的进度。
3. **Workflow System 有一个数据传递模块。**它负责 tasks 之间信息的传递，这些信息可大可小，可以通过Pod Annotation暂存信息，也可能通过对象存储系统暂存信息。

## Workflow System 的应用场景？

1. CICD
2. MapReduce

## 同类型的产品？

1. tekton  这是从 knative 中剥离出来的项目
2. Fission Workflow 是基于它的 Fission serverless 项目而构建的。
   1. Joe 不怎么看好 Fission Workflow。
   2. Fission 的一个特点是可以支持 Pod 的热启动，这要求 Fission 的 pool manager 需要有很大的权限，所有的 CRD 和 Pod（workload） 都需要在同一个namespace中。
   3. 扩展性不太好。比如，你无法实现多租户的功能。
3. Brigade
   1. 与 git repo 深度集成，但无法用于CICD
   2. 可以动态地、增量地构建 workflow，但是却无法查看 workflow的全貌
   3. 支持 trigger。Argo 需要通过 Argo CI 来触发 argo workflow 的执行

## Argo Workflow 的当前状态 （8.4k，v3.0.2）？

### 关于 CRD

1. 通过 CRD 描述了 workflow 的组成，
2. 通过 Mini language 可以实现 IF/ELSE 逻辑。即任务之间的依赖关系，移动动态运行的结果。
3. 作者觉得这种 Mini Language 可能会不受控制，后面会变得复杂起来。

### 关于输入和输出

1. 对于小文件或者小数据，可以通过 Pod Annotation 来存储。获取Container A的输出并将它写到Pod的Annotation当中，这是由 Side Car 容器来完成的。
   1. 你在 submit 一个任务时，需要使用--serviceaccount 指定一个有权限写 Annotation 的 Service Account。
   2. 作者认为这里比较好的一个设计：argo controller 的权限与workflow 中 pod 的权限的分离。用户可以控制 argo workflow 的权限。
2. 如何在 pod 之间处理大文件？
   1. 需要借助一个 Object Storage。了解下 minio（27k）。
      1. 你可以将本地磁盘，通过 minio，对外暴露成 S3 存储
      2. 你可以将块存储，通过 minio，对外暴露成S3 存储。
   2. 这是一个通用问题。作者期望的是，每个服务的运行，都已经默认绑定好了一个对应的 Object Store，无需进行认证。Borg 是这样做的。
3. PVC 同时只能绑定到一个Pod上。

## Minio Server

1. 安装 minio server。`docker run -p 9000:9000 minio/minio server /data`。
   1. 安装成功以后，通过访问：http://127.0.0.1:9000 即可通过UI界面访问 minio server
   2. 你可以创建 bucket，上传文执行。相应的，在容器内部，有一个目录 /data。bucket 对应到 /data 下的一个子目录，上传的文件对应 /data/xxx 下的一个文件。
   3. 你可以使用 `-v /mnt/data:/data` 参数，将bucket、file持久化到宿主机上。
2. Start single node server with 5 local drives `docker run -v /tmp/data:/data minio/minio server /data/data{1...5}`。如此，minio server 会将上传的文件进行 sharding，分片存储到每个driver上。
3. minio server 对外暴露的接口完全兼容 AWS S3 的标准，所以你可以直接使用 s3cmd 来安装创建bucket、创建文件。

## Minio Client

1. 安装 minio client。`brew install minio/stable/mc`。它的配置文件在`~/.mc/config.json`。
2. 它是一个客户端，与minio server 没有什么关系。对它的定位如下：MinIO Client (mc) provides a modern alternative to UNIX commands like ls, cat, cp, mirror, diff, find etc。It supports filesystems and Amazon S3 compatible cloud storage service (AWS Signature v2 and v4). 所以，你可以将minio client 作为 s3cmd 的替代品。
3. 修改 local 配置，使得它能够访问 mino server。执行 `mc ls local` 进行验证.

```json
{
	"version": "10",
	"aliases": {
		"gcs": {
			"url": "https://storage.googleapis.com",
			"accessKey": "YOUR-ACCESS-KEY-HERE",
			"secretKey": "YOUR-SECRET-KEY-HERE",
			"api": "S3v2",
			"path": "dns"
		},
		"local": {
			"url": "http://localhost:9000",
			"accessKey": "minioadmin",   # 默认自带的 access key
			"secretKey": "minioadmin",   # 默认自带的 secret key
			"api": "S3v4",
			"path": "auto"
		},
		"play": {
			"url": "https://play.min.io",
			"accessKey": "Q3AM3UQ867SPQQA43P2F",
			"secretKey": "zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG",
			"api": "S3v4",
			"path": "auto"
		},
		"s3": {
			"url": "https://s3.amazonaws.com",
			"accessKey": "YOUR-ACCESS-KEY-HERE",
			"secretKey": "YOUR-SECRET-KEY-HERE",
			"api": "S3v4",
			"path": "dns"
		}
	}
}
```