# Deploy MinIO and YS1000 in offline environment

## 内容索引

- [1. 运行环境与文件准备](#1-运行环境与文件准备)
    - [1.1 检查集群环境与连接](#11-检查集群环境与连接)
    - [1.2 拷贝文档并上传镜像](#12-拷贝文档并上传镜像)
- [2. 安装Helm](#2-安装Helm)
- [3. 创建storageclass](#3-创建storageclass)
- [4. 部署MinIO](#4-部署MinIO)
    - [4.1 通过helm安装minio](#41-通过helm安装minio)
    - [4.2 MinIO配置用户和bucket](#42-MinIO配置用户和bucket)
- [5. 部署YS1000](#5-部署YS1000)


## 1. 运行环境与文件准备

这套方案是基于k8s集群的部署，因此需要一个能够连接k8s集群并有root权限的用户，以及一个私有镜像仓库，把所有需要的文件拷贝到集群所在机器，并将相关image推到私有镜像仓库以方便后续运维。

### 1.1 检查集群环境与连接

检查以下环境准备

|Requirment|Example|
|:--|:--|
|k8s|v1.15.4|
|docker|v19.03.8|
|repositry|registry.cn-shanghai.aliyuncs.com|

```
# kubectl get nodes
NAME        STATUS   ROLES    AGE    VERSION
bj-demo-2   Ready    master   119d   v1.15.4

# docker version
Client: Docker Engine - Community
 Version:           19.03.8

# docker login registry.cn-shanghai.aliyuncs.com
```

### 1.2 拷贝文档并上传镜像

第一步，使用scp或u盘传输文档offline-pak到集群所在机器并检查文件。

```
# cd offline-pak/
# ll
总用量 384948
-rw-r--r-- 1 root root  13861119 12月 23 11:30 helm-v3.7.0-linux-amd64.tar.gz
-rw-r--r-- 1 root root    307712 12月 15 20:03 minio-9.2.5.tar
-rw------- 1 root root 131659264 12月 23 11:36 minio-client.tar
-rw------- 1 root root 248294912 12月 23 11:34 minio.tar
-rw-r--r-- 1 root root     41688 12月 12 14:37 minio-values.yaml
-rwxrwxrwx 1 root root      2498 12月 23 14:53 setup-image.sh
drwxr-xr-x 2 root root      4096 12月 23 12:57 ys1000

# cd ys1000/
# ll
总用量 2590960
-rw------- 1 root root 896296960 12月 23 12:46 hook-runner.tar
-rw------- 1 root root 171201536 12月 23 11:58 mig-controller.tar
-rw------- 1 root root 171201536 12月 23 12:21 mig-discovery.tar
-rw------- 1 root root 603714048 12月 23 12:16 mig-ui.tar
-rw-r--r-- 1 root root     17028 12月 23 12:46 qiming-operator-2.1.0.tgz
-rw------- 1 root root 149760000 12月 23 11:53 qiming-operator.tar
-rw-r--r-- 1 root root      3986 12月 23 12:46 qiming-values.yaml
-rw------- 1 root root 327838208 12月 23 12:55 velero-installer.tar
-rw------- 1 root root  67578368 12月 23 12:57 velero-plugin-for-aws.tar
-rw------- 1 root root  73300480 12月 23 12:00 velero-plugin-for-csi.tar
-rw------- 1 root root  84515328 12月 23 11:49 velero-restic-restore-helper.tar
-rw------- 1 root root 107690496 12月 23 13:00 velero.tar
```

第二步，将私有镜像仓库的地址配置完后，跑脚本setup-image.sh，导入MinIO和YS1000的镜像，并修改tag再上传到私有仓库。

```
# cd offline-pak/
# export REPOSITRY_ID=registry.cn-shanghai.aliyuncs.com/ys1000/
# ./setup-image.sh 
Loaded image: bitnami/minio:2021.12.10-debian-10-r0
801dfff41078: Loading layer [==================================================>]   80.9kB/80.9kB
...
echo IMAGE PUSH SUCCESSFULLY!
```


## 2. 安装Helm

第一步，解压helm-v3.7.0-linux-amd64.tar.gz

```
# tar -zxvf helm-v3.7.0-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
```

第二步，将二进制文件移至bin目录后查看helm命令。

```
# mv linux-amd64/helm /usr/local/bin/helm

# helm version
version.BuildInfo{Version:"v3.7.0", GitCommit:"eeac83883cb4014fe60267ec6373570374ce770b", GitTreeState:"clean", GoVersion:"go1.16.8"}
```

## 3. 创建storageclass

第一步，解压helm-v3.7.0-linux-amd64.tar.gz

```
# 
```

## 4. 部署MinIO

本文中我们将使用已经打包好的helm文件和docker image通过替换minio-values.yaml的参数实现私有化部署。

### 4.1 通过helm安装minio

第一步，修改minio-value.yaml 中的值，替换成上一步新建的storageclass和私有镜像仓库的地址，其他参数如resources等根据需求替换（此处为默认值）。
|Key|Value|
|:--|:--|
|global.storageClass|managed-nfs-storage|
|image.registry|registry.cn-shanghai.aliyuncs.com|
|image.repository|ys1000/minio|
|image.tag|2021.12.10|
|clientImage.registry|registry.cn-shanghai.aliyuncs.com|
|clientImage.repository|ys1000/minio-client|
|clientImage.tag|2021.12.10|
|resources.limits.cpu|100m|
|resources.limits.memory|64Mi|

```
# kubectl get storageclasses
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  127d

# cat minio-values.yaml
...
  storageClass: managed-nfs-storage

image:
  registry: registry.cn-shanghai.aliyuncs.com
  repository: ys1000/minio
  tag: 2021.12.10
clientImage:
  registry: registry.cn-shanghai.aliyuncs.com
  repository: ys1000/minio-client
  tag: 2021.12.10
```

第二步，使用helm本地安装minio。

```
# helm install minio minio-9.2.5.tgz --namespace minio --create-namespace -f minio-values.yaml
```

第三步，按安装完minio后的实际输出命令（每次输出不同，不能直接复制本文！），继续安装minio-client。

```
export ROOT_USER=$(kubectl get secret --namespace minio minio-1639291135 -o jsonpath="{.data.root-user}" | base64 --decode)
export ROOT_PASSWORD=$(kubectl get secret --namespace minio minio-1639291135 -o jsonpath="{.data.root-password}" | base64 --decode)
kubectl run --namespace minio minio-1639462625-client \
     --rm --tty -i --restart='Never' \
     --env MINIO_SERVER_ROOT_USER=$ROOT_USER \
     --env MINIO_SERVER_ROOT_PASSWORD=$ROOT_PASSWORD \
     --env MINIO_SERVER_HOST=minio-1639462625 \
     --image docker.io/bitnami/minio-client:2021.12.10-debian-10-r1 -- admin info minio
```

第四步，等待两个pod起来后，修改minio的service访问方式为nodeport。

```
# kubectl get pod -n minio
NAME                                READY   STATUS      RESTARTS   AGE
minio-1639462625-7bcfb6b999-bblth   1/1     Running     0          9m
minio-1639462625-client             0/1     Completed   0          3m

# kubectl -n minio edit svc minio

...
spec:
  clusterIP: 10.98.207.210
  > clusterIPs:
  > - 10.98.207.210
  > externalTrafficPolicy: Cluster
  ports:
  - name: minio-api
    > nodePort: 31900
    port: 9000
    protocol: TCP
    targetPort: minio-api
  - name: minio-console
    > nodePort: 31901
    port: 9001
    protocol: TCP
    targetPort: minio-console
  selector:
    app.kubernetes.io/instance: minio-1639291135
    app.kubernetes.io/name: minio
  sessionAffinity: None
  > type: NodePort

# kubectl -n minio get svc minio
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
minio   NodePort   10.98.207.210   <none>        9000:31900/TCP,9001:31901/TCP   42h
```

### 4.2 MinIO配置用户和bucket

第一步，在minio-client中查看用户名和密码，前端登陆minio。
```
# kubectl -n minio get pod minio-client -o yaml
...
env:
    - name: MINIO_SERVER_ROOT_USER
      value: admin
    - name: MINIO_SERVER_ROOT_PASSWORD
      value: Rx7v4dYVPxPdioxrZCfjjmmY4bxHQYBMkffs1oW6
```
第二步，浏览器打开

http:// < cluster ip > :31901/login

输入用户名：admin，密码：Rx7v4dYVPxPdioxrZCfjjmmY4bxHQYBMkffs1oW6。

第三步，点击左侧导航栏Users，创建一个user，记录access key和secret key（请保存好这对key）, 并选择权限大于等于readwrite。
access key：minio
secret key：minio123

第四步，点击左侧导航栏Buckets，创建一个bucket：test。


## 5. 部署YS1000

第一步，进入/ys1000文件夹，修改qiming-value.yaml 中的值替换成私有镜像仓库的repositry。

```
# cd ys1000/

# cat qiming-values.yaml
...
image:
  repository: registry.cn-shanghai.aliyuncs.com/ys1000/qiming-operator
  pullPolicy: Always
  tag: "v2.1.0"

componentImages:
  uiImage:
    repository: registry.cn-shanghai.aliyuncs.com/ys1000/mig-ui
    tag: "v2.1.0"
  discoveryImage:
    repository: registry.cn-shanghai.aliyuncs.com/ys1000/mig-discovery
    tag: "v2.1.0"
  migControllerImage:
    repository: registry.cn-shanghai.aliyuncs.com/ys1000/mig-controller
    tag: "v2.1.0"
  resticHelperImage:
    repository: registry.cn-shanghai.aliyuncs.com/ys1000/velero-restic-restore-helper
    tag: "v1.7.0"
  veleroInstallerImage:
    repository: registry.cn-shanghai.aliyuncs.com/ys1000/velero-installer
    tag: "v2.1.0"
  hookRunnerImage:
    repository: registry.cn-shanghai.aliyuncs.com/ys1000/hook-runner
    tag: "latest"
velero:
  enabled: true
  image: registry.cn-shanghai.aliyuncs.com/ys1000/velero:v1.7.0
  plugins: registry.cn-shanghai.aliyuncs.com/ys1000/velero-plugin-for-aws:v1.3.0,registry.cn-shanghai.aliyuncs.com/ys1000/velero-plugin-for-csi:v0.2.0
```

第二步，使用helm本地安装YS1000。

```
# helm install qiming-operator qiming-operator-2.1.0.tgz --namespace qiming-migration --create-namespace -f qiming-values.yaml --set service.type=NodePort --set s3Config.provider=aws --set s3Config.name=minio --set s3Config.accessKey=minio --set s3Config.secretKey=minio123 --set s3Config.bucket=test --set s3Config.s3Url=http://139.198.27.211:31900
```

第三步，查看qiming-operator版本和pod运行情况，等待pod就绪

```
# kubectl -n qiming-migration get pods
NAME                                                READY   STATUS    RESTARTS   AGE
mig-controller-default-5b4f996675-m86hw             1/1     Running   0          22s
qiming-operator-77c4cb9f76-jd2tq                    1/1     Running   0          35s
qiming-operator-velero-installer-6867d55f7d-przjh   1/1     Running   0          35s
ui-discovery-default-7cc4bb646-bd5dr                2/2     Running   0          22s

# kubectl -n qiming-backend get pods
NAME                      READY   STATUS    RESTARTS   AGE
restic-nxb4k              1/1     Running   0          48s
restic-vtjvf              1/1     Running   0          48s
velero-58d86bfd96-7qbmf   1/1     Running   0          48s

# helm list -A
NAME            NAMESPACE               REVISION        UPDATED                                 STATUS          CHART                   APP VERSION 
qiming-operator qiming-migration        1               2021-12-22 11:03:12.138661316 +0800 CST deployed        qiming-operator-2.1.0   2.1.0
```

第四步，安装成功后根据提示获取访问url和token，并登陆YS1000前端

```
2. After status is ready, get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace qiming-migration -o jsonpath="{.spec.ports[0].nodePort}" services ui-service-default )
  export NODE_IP=$(kubectl get nodes --namespace qiming-migration -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

3. Login web UI with the token by running these commands:
  export SECRET=$(kubectl -n qiming-migration get secret | (grep qiming-operator |grep -v helm || echo "$_") | awk '{print $1}')
  export TOKEN=$(kubectl -n qiming-migration describe secrets $SECRET |grep token: | awk '{print $2}')
  echo $TOKEN
```