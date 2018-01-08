---
layout: post
title:  Kubernetes静态持久卷的探索学习
categories: [docker,Cloud]
date: 2017-01-11 10:58:30 +0800
keywords: [docker,云计算,kubernetes]
---

>随着docker及Kubernetes技术发展的越来越成熟稳定，容器平台不仅仅局限于部署无状态应用，越来越多的有状态服务也可以在容器云上稳定地部署运行，本文主要就讲讲kubernetes中的PersistentVolume特性（静态PV）。

### 1 名词概念

**Volume**

Volume是Pod的挂载接口，生命周期同Pod，可以在Pod内的各个Container之间进行共享，主要用于存储Pod生命周期内的临时数据，当然，也可以挂在在Host主机或者其他后端存储介质上实现永久存储，根据选用的Volume Type可以实现不同的存储需求，下边是Volume支持的类型：

- emptyDir
- hostPath
- gcePersistentDisk
- awsElasticBlockStore
- nfs
- iscsi
- flocker
- glusterfs
- rbd
- cephfs
- gitRepo
- secret
- persistentVolumeClaim
- downwardAPI
- azureFileVolume
- azureDisk
- vsphereVolume
- Quobyte

此处不作一一介绍，可以参考文档：[Volume Docs](https://kubernetes.io/docs/user-guide/volumes/)。

**PersistentVolume**（PV）

假如有一个独立的存储后端，底层实现可以是NFS、GlusterFS、Cinder、HostPath等等，可以使用PV从中划拨一部分资源用于kubernetes的存储需求，其生命周期不依赖于Pod，是一个独立存在的虚拟存储空间，但是不能直接被Pod的Volume挂载，此时需要用到PVC。

PV支持的后端存储插件：

- GCEPersistentDisk
- AWSElasticBlockStore
- AzureFile
- AzureDisk
- FC (Fibre Channel)
- NFS
- iSCSI
- RBD (Ceph Block Device)
- CephFS
- Cinder (OpenStack block storage)
- Glusterfs
- VsphereVolume
- HostPath (single node testing only – local storage is not supported in any way and WILL NOT WORK in a multi-node cluster)

**PersistentVolumeClaim**（PVC）

Pod使用PV资源是通过PVC来实现的，PVC可以理解为资源使用请求，一个Pod需要先明确使用的资源大小、访问方式，创建PVC申请提交到kubernetes中的PersistentVolume Controller，由其调度合适的PV来与PVC绑定，然后Pod中的Volume就可以通过PVC来使用PV的资源。

**StorageClasse**

用于定义动态PV资源调度，相比起静态PV资源来说，动态PV不需要预先创建PV，而是通过PersistentVolume Controller动态调度，根据PVC的资源请求，寻找StorageClasse定义的符合要求的底层存储来分配资源。

### 2 创建PersistentVolume

定义`PersistentVolume`，这里使用`hostPath`作为存储底层。

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv001
  labels:
    release: stable
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /tmp/data
```

也可以使用NFS或者其他插件作为存储底层，需要提前准备好NFS Server：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
labels:
  release: stable
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /tmp/data
    server: 172.17.0.2
```

**capacity**

用于定义PV的存储容量，当前只支持定义大小，未来会实现其他能力如：IOPS、吞吐量。

**accessModes**

用于定义资源的访问方式，受限于存储底层的支持，访问方式包括以下几种：

- ReadWriteOnce – 被单个节点mount为读写rw模式
- ReadOnlyMany  – 被多个节点mount为只读ro模式
- ReadWriteMany – 被多个节点mount为读写rw模式

下边途中列举了k8s支持的存储插件的访问方式：

|    Volume Plugin     | ReadWriteOnce | ReadOnlyMany | ReadWriteMany |
|----------------------|---------------|--------------|---------------|
| AWSElasticBlockStore | x             | –            | –             |
| AzureFile            | x             | x            | x             |
| CephFS               | x             | x            | x             |
| Cinder               | x             | –            | –             |
| FC                   | x             | x            | –             |
| FlexVolume           | x             | x            | –             |
| GCEPersistentDisk    | x             | x            | –             |
| Glusterfs            | x             | x            | x             |
| HostPath             | x             | –            | –             |
| iSCSI                | x             | x            | –             |
| NFS                  | x             | x            | x             |
| RDB                  | x             | x            | –             |
| VsphereVolume        | x             | –            | –             |

**persistentVolumeReclaimPolicy**

用于定义资源的回收方式，也首先与存储底层的支持，现有的回收策略：

- Retain  – 手动回收
- Recycle – 删除数据 (“rm -rf /thevolume/*”)
- Delete  – 通过存储后端删除卷，后端存储例如AWS EBS, GCE PD或Cinder等。

目前只有NFS和HostPath支持Recycle策略，AWS EBS、GCE PD、Azure Disk、Cinder支持Delete策略。

>注意：Recycle策略会通过运行一个busybox容器来执行数据删除命令，默认定义的busybox镜像是：`gcr.io/google_containers/busybox:latest`，并且`imagePullPolicy: Always`，如果需要调整配置，需要增加kube-controller-manager 启动参数：`--pv-recycler-pod-template-filepath-hostpath=/etc/kubernetes/manifests/recycler.yml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler-
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: [Path of Persistent Volume hosted]
  containers:
  - name: pv-recycler
    image: "gcr.io/google_containers/busybox"
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

### 3 创建PersistentVolumeClaim

定义`PersistentVolumeClaim`。

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      release: stable
```

**accessModes**

与PersistentVolume的访问方式一致，PersistentVolume Controller调度访问方式一致PV资源与PVC绑定。

**resources**

用于定义申请使用的存储资源大小，适用于kubernetes的resource模型，具体信息可以查看[Resource Model docs](https://github.com/kubernetes/kubernetes/blob/master/docs/design/resources.md)。

**selector**

定义PVC申请过滤PV卷集，搭配label定义使用，同kubernetes中其他的selector概念一致，用法上稍有不同，增加了匹配选项：

- matchLabels – 匹配标签，卷标签必须匹配某个值
- matchExpressions – 匹配表达式，由键值对，操作符构成，操作符包括 In，NotIn，Exists，和 DoesNotExist。

此外还有`volume.beta.kubernetes.io/storage-class`定义，具有相同定义的PV和PVC才会绑定，具体用法可以查看[PersistentVolume docs](https://kubernetes.io/docs/user-guide/persistent-volumes/)。

### 4 挂载Volume到Pod

PV和PVC创建并绑定之后，类似这样：

```Bash
NAME       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM               REASON    AGE
pv/pv001   5Gi        RWO           Recycle         Bound     default/myclaim-1             11m

NAME            STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
pvc/myclaim-1   Bound     pv001     5Gi        RWO           3s
```

PersistentVolume有四种状态：

- Available – 可用状态
- Bound     – 绑定到PVC
- Released  – PVC被删掉，但是尚未回收
- Failed    – 自动回收失败

挂载创建好的PVC：myclaim-1到Pod上：

```Bash
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: dockerfile/nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim-1
```

挂载成功后，Pod所在的Host上会自动创建`/tmp/data`用于存储数据，HostPath Volume便于测试调试，但是只适用于单节点环境，多节点环境中如果Pod漂移或者重建后不在原先节点，则无法访问原来的数据。

静态持久卷的探索学习就到这里，后边会再写一篇专门介绍动态持久卷的文章，请持续关注。

>版权声明：允许转载，请注明原文出处：http://cloudnil.com/2017/01/11/Static-PersistentVolume/。