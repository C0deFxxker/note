---
title: "k8s 存储"
category: k8s
---
# K8s存储概述
## Persistent volumes(PV)
下面是NFS的PV配置：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-1
  labels:
    release: stable
    capacity: 100Gi
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessMode:
  - ReadWriteOnece
  - ReadOnlyMany
  storageClassName: normal
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /tmp
    server: 127.17.0.8
```

字段解析：
* capacity: 指定这个PV的总容量。
* volumeMode: 可选值有"Filesystem"和"Block"，默认是"Filesystem"。
* accessMode: 可选值有"ReadOnlyMany","ReadWriteOnce","ReadWriteMany"。Once指的是同时只能被一个节点操作，Many指的是同时可被多个节点操作。
* persistentVolumeReclaimPolicy: 指定一个PVC被删除时当前PV需要执行的操作。可选值有"Retain","Delete","Recycle"。不同卷类型的PV可选的值稍微有些不同，比如只有NFS与HostPath模式提供"Recycle"策略。
* storageClassName: 这个字段是可选值，字符串类型。只有带有相同storageClassName的PVC才能索取这个PV，并且需要注意的是若当前PV没有设置storageClassName字段值，则只有那些同样没有设置storageClassName字段值的PVC能索取这个PV。
* 卷类型：卷类型可选值有很多，将会在后续注意说明。设置的方式是把卷类型名称（如nfs）作为spec下的一个key，对应的值类型是对象，美中卷类型对象都有不同的字段。

## Persistent volume claims(PVC)
生成PV后还不能立刻应用到容器中，需要创建一个PVC对象来配置对这个PV的索取方式，例子如下：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-claim
spec:
  accessMode:
  - ReadWriteOnce
  resources:
    requests:
      storage: 80Gi
  storageClassName: normal
  selector:
    matchLabels:
      release: stable
    matchExpressions:
    - {key: capacity, operator: In, values: [80Gi, 1000Gi]}
```

PVC匹配PV的方式并不是使用name指定，而是通过storageClassName, capacity, selector字段进行筛选匹配。PVC与PV都是命名空间域下的资源，PVC只能索取相同命名空间下的PV。

字段解析：
* accessMode: TODO。
* resources: 指定申请PV的空间大小，上面指定了80Gi，若是向100Gi的PV索取则剩下的20Gi将会浪费掉。
* storageClassName: 上一节提到过，这个PVC只能索取带有相同storageClassName的PV。
* selector: 相当于PV的过滤器，matchLabels指定只能索取带有这些标签的PV。matchExpressions可以指定筛选表达式，例子中筛选容量为80Gi或100Gi的PV。假设实际中还存在200Gi或500Gi的PV，而这个只需要80Gi的PVC去申请了这些PV将会是极大的资源浪费，通过这个selector的筛选可以禁止这个PVC索取这些大容量的PV。但即使没有这个selector的限制，K8s还是会索取现有容量最小且能满足PVC申请容量的PV。

生成了PVC后，创建部署时就可以通过PVC的“名称”来挂载卷到容器中，例子如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: the-pod
spec:
  containers:
  - name: the-container
    image: some-image
    volumeMounts:
    - mountPath: /mnt/data
      name: persistent-volume
  volumes:
  - name: persistent-volume
    persistentVolumeClaim:
      claimName: storage-claim
```

PVC的卷就会挂载到容器中的 /mnt/data 目录下。

## 块存储
Kubernetes v1.9 新增了 Alpha 版的 Raw Block Volume，使用前需要为 kube-apiserver、kube-controller-manager 和 kubelet 开启 BlockVolume 特性，即添加命令行选项 --feature-gates=BlockVolume=true。

块存储为应用程序提供了直接对底层存储设备进行操作的方式而不用经过文件系统抽象层接口，这样可以达到更高的存储效率。

支持块存储的 PV 插件包括：
* Local Volume
* fc
* iSCSI
* Ceph RBD
* AWS EBS
* GCE PD
* AzureDisk
* Cinder

下面给出块存储的例子：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnece
  volumeMode: Block # 块存储
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessMode:
  - ReadWriteOnce
  volumeMode: Block # 块存储
  resources:
    requests:
      storage: 10Gi
```

块存储在机器上应该是以“设备”形式呈现，不同于文件系统的目录形式。部署应用时需要指定块设备同步到容器中 /dev 目录下的一个设备文件。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
  - name: fc-container
    image: fedora:26
    command: ["/bin/sh", "-c"]
    args: ["tail -f /dev/null"]
    volumeDevices:  # 块存储以设备文件形式存放到容器中的 /dev/xvda
    - name: data
      devicePath: /dev/xvda
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: block-pvc
```

# 各种卷类型
K8s提供的卷类型非常丰富，包括（[详情参考官网](https://kubernetes.io/docs/concepts/storage/)）：
* awsElasticBlockStore
* azureDisk
* azureFile
* cephfs
* cinder
* configMap
* csi
* downwardAPI
* emptyDir
* fc (fibre channel)
* flexVolume
* flocker
* gcePersistentDisk
* gitRepo (deprecated)
* glusterfs
* hostPath
* iscsi
* local
* nfs
* persistentVolumeClaim
* projected
* portworxVolume
* quobyte
* rbd
* scaleIO
* secret
* storageos
* vsphereVolume

接下来将详细介绍部分存储类型的使用。

## emptyDir
用于同一个Pod下不同容器之间共用的卷，当Pod被销毁时都会删除旧卷，这意味着emptyDir卷不能持久化Pod的信息，只用于Pod中不同容器之间以文件系统的方式通信。

emptyDir的存储媒介可以选择：内存或磁盘。使用内存作为存储媒介时会以tmpfs类型的卷挂载到容器目录下，该目录下的数据都会保存在宿主机的内存中。使用磁盘作为存储媒介时会将会挂载宿主机 "/var/lib/kubelet/pods/**{podUid}**/volumes/kubernetes.io~empty-dir/**{volumeName}**" 目录到容器中。

下面给出emptyDir的例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: test-emptypath
  name: test-emptypath
spec:
  containers:
    - name: emptydir-producer
      image: busybox
      volumeMounts:
       - name: log-storage
         mountPath: /home/log/
      command:
      - /bin/sh
      - -c
      - "while [ true ]; do echo $(date) >> /home/log/test.log; sleep 5; done"
    - name: emptydir-consumer
      image: busybox
      volumeMounts:
       - name: log-storage
         mountPath: /home/log/
      command:
      - tail
      - -f
      - /home/log/test.log
  volumes:
  - name: log-storage
    emptyDir:
      # medium: Memory    # 卷媒介采用内存存储，不指定这个属性则表示采用磁盘作为存储媒介
      sizeLimit: 300Mi  # 卷容量为300Mb
```

这个Pod中包含了两个容器，第一个容器用于产生日志输出到emptydir卷目录(/home/log)下的test.log文件中，另一个容器负责读取emptydir卷目录下的test.log文件内容，采用磁盘作为卷存储媒介，限制卷容量为300Mb。

先进入第一个容器中确认容器中的日志已输出/home/log/test.log文件中。

```sh
$ kubectl exec test-emptypath -c emptydir-producer cat /home/log/test.log
Wed Jan 1 15:21:21 UTC 2020
Wed Jan 1 15:21:26 UTC 2020
Wed Jan 1 15:21:31 UTC 2020
Wed Jan 1 15:21:36 UTC 2020
Wed Jan 1 15:21:41 UTC 2020
# 确认emptydir挂载到/home/log目录下，并且是以磁盘作为存储媒介
$ kubectl exec test-emptypath -c emptydir-producer df
Filesystem           1K-blocks      Used Available Use% Mounted on
overlay              478375712  69699844 408675868  15% /
tmpfs                    65536         0     65536   0% /dev
tmpfs                 32695936         0  32695936   0% /sys/fs/cgroup
/dev/mapper/centos-root 478375712  69699812 408675900  15% /dev/termination-log
/dev/mapper/centos-root 478375712  69699812 408675900  15% /home/log
/dev/mapper/centos-root 478375712  69699812 408675900  15% /etc/resolv.conf
/dev/mapper/centos-root 478375712  69699812 408675900  15% /etc/hostname
/dev/mapper/centos-root 478375712  69699812 408675900  15% /etc/hosts
shm                      65536         0     65536   0% /dev/shm
tmpfs                 32695936        12  32695924   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                 32695936         0  32695936   0% /proc/acpi
tmpfs                    65536         0     65536   0% /proc/kcore
tmpfs                    65536         0     65536   0% /proc/keys
tmpfs                    65536         0     65536   0% /proc/timer_list
tmpfs                    65536         0     65536   0% /proc/timer_stats
tmpfs                    65536         0     65536   0% /proc/sched_debug
tmpfs                 32695936         0  32695936   0% /proc/scsi
tmpfs                 32695936         0  32695936   0% /sys/firmware
```

然后查看第二个容器的输出日志确实得到同样的内容。

```sh
$ kubectl logs -f test-emptypath emptydir-consumer
Wed Jan 1 15:21:21 UTC 2020
Wed Jan 1 15:21:26 UTC 2020
Wed Jan 1 15:21:31 UTC 2020
Wed Jan 1 15:21:36 UTC 2020
Wed Jan 1 15:21:41 UTC 2020
```

接着我们尝试在宿主机查看这个emptydir卷对应的目录文件。
```sh

```

## hostPath
## local
## secret
## configMap
## persistentVolumeClaim
一个persistentVolumeClaim卷需要绑定一个PersistentVolume后才能被Pod挂载。PersistentVolumes提供存储的抽象概念，它记录复杂的存储细节（使用的存储驱动及相关配置信息）。这样一来用户只需直接与persistentVolumeClaim卷打交道，无需了解后面绑定PersistentVolumes的存储细节。

上面已对PVC做过详细介绍，这里不在阐述。
## glusterFS

## cephRBD
Ceph RBD是基于Ceph的Block类型存储模式，优点是应用直接调用底层存储设备能力，不需要经过文件系统抽象接口，存取效率更高。缺点是只能挂载ReadWriteOnce模式的PVC，不适合做多应用文件共享。

## cephFS
## projected