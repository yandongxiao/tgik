[TOC]

# Velero - Backup & Restore Kubernetes Cluster

## 简介

![image-20210719215120261](https://raw.githubusercontent.com/yandongxiao/typera/main/img/image-20210719215120261.png)

## Running minio container

velero 需要使用 S3 bucket 作为后端存储。

1. 创建 minio service

   ```bash
   # 登录 http://127.0.0.1:9000 验证 minio servie 安装成功
   docker pull minio/minio
   docker run \
     --name minio \
     -p 9000:9000 \
     -p 9001:9001 \
     -v data:/data \
     -e "MINIO_ROOT_USER=AKIAIOSFODNN7EXAMPLE" \
     -e "MINIO_ROOT_PASSWORD=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" \
     minio/minio server /data --console-address ":9001"
   ```

2. 生成 AS、SK。以 docker 启动 minio 时，会在下面的文件中自动生成AK、SK。

   ```bash
   docker exec -it minio cat /data/.minio.sys/config/config.json | egrep "(access|secret)_key"
   
   # Change access key and secret key from the Minio dashboard.
   # 登录 http://127.0.0.1:9000，可以修改 AK、SK的值
   # 重新登录
   ```

3. 创建 bucket

   ```bash
   # 登录 http://127.0.0.1:9000
   # 创建 bucket
   ```

## Download Velero 1.0.0 Release

```bash
# 这是 linux 版本
wget https://github.com/heptio/velero/releases/download/v1.0.0/velero-v1.0.0-linux-amd64.tar.gz
tar zxf velero-v1.0.0-linux-amd64.tar.gz
sudo mv velero-v1.0.0-linux-amd64/velero /usr/local/bin/
rm -rf velero*

# 运行 velero 命令，会发现找不到 velero server
# 这也是 velero 的运行方式：Client + Server
Client:
	Version: v1.6.1
	Git commit: ef34b9b6546b0e972eddce71a40cf14a7b9eb393
<error getting server version: no matches for kind "ServerStatusRequest" in version "velero.io/v1">
```

## Create credentials file (Needed for velero initialization)

```bash
# minio.credentials 可以是任意名称
cat <<EOF>> minio.credentials
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
```

## Install Velero in the Kubernetes Cluster

```bash
# minio 完全兼容 aws s3
# 在 Kubernetes 的 velero namespace 中，创建 Deployment
velero install \
   --plugins velero/velero-plugin-for-aws:v1.0.0 \
   --provider aws \
   --bucket kubedemo \
   --secret-file ./minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://10.242.9.31:9000
   
# Velero is installed! ⛵ Use 'kubectl logs deployment/velero -n velero' to view the status.

➜  ~ kubectl -n velero get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/velero-b5dd4f9cb-b7nt9   1/1     Running   0          73s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velero   1/1     1            1           73s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/velero-b5dd4f9cb   1         1         1       73s   
```

## Enable tab completion for preferred shell

```
source <(velero completion zsh)
```



### velero 的备份能力

1. 备份整个 Cluster
2. 备份整个 Namespace
3. 备份某个指定的资源
4. 你也可以基于排除法进行备份

## 参考

1. https://github.com/justmeandopensource/kubernetes/blob/master/docs/setup-velero-notes.md