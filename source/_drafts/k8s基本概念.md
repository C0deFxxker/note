---
title: "k8s基本概念"
category: k8s
mathjax: false
---
本章介绍K8s核心概念，了解K8s的基本组件及其作用，为后续深入学习做铺垫。

<!--more-->

# 资源
## Cluster
K8s集群，由上图中的所有组件组成，包含了计算、存储、网络资源。

## Node
* Node：Node指定一台宿主机，可以是物理机或是虚拟机。K8s中节点分为两种：Master Node 与 Worker Node。
* Master Node：对集群做出全局性决策(例如：调度)，以及检测和响应集群事件(副本控制器的replicas字段不满足时,启动新的副本)。包括组件有：kube-apiserver,kube-control-manager,kube-schduler。
* Worker Node：落实容器生命周期管理工作，以及POD的路由转发处理工作。包括组件有：kubelet,kube-proxy。

## Pod
Pod可以理解为一个虚拟节点，Kubernetes以Pod为最小单元进行调度、扩展、资源共享、管理生命周期。每个Pod中可以包含一个或多个容器，Pod中的所有容器都会共享同一个网络以及文件系统，也就是说Pod中的多个容器不能同时占用同一个Pod端口，K8s会为每个Pod创建一个volume，同一个Pod下多个容器的文件共享就是通过挂载同一个volume来实现的。同一个Pod中的所有容器只会运行在同一Node上。

## ReplicationController
RC是K8s集群中最早的保证Pod高可用的API对象。通过监控运行中的Pod来保证集群中运行指定数目的Pod副本。指定的数目可以是多个也可以是1个；少于指定数目，RC就会启动运行新的Pod副本；多于指定数目，RC就会杀死多余的Pod副本。即使在指定数目为1的情况下，通过RC运行Pod也比直接运行Pod更明智，因为RC也可以发挥它高可用的能力，保证永远有1个Pod在运行。RC是K8s较早期的技术概念，只适用于长期伺服型的业务类型，比如控制小机器人提供高可用的Web服务。

## ReplicaSet
RS是新一代RC，提供同样的高可用能力，区别主要在于RS后来居上，能支持更多种类的匹配模式。副本集对象一般不单独使用，而是作为Deployment的理想状态参数使用。

## Deployment
部署表示用户对K8s集群的一次更新操作。部署是一个比ReplicaSet应用模式更广的API对象，可以是创建一个新的服务，更新一个新的服务，也可以是滚动升级一个服务。滚动升级一个服务，实际是创建一个新的RS，然后逐渐将新RS中副本数增加到理想状态，将旧RS中的副本数减小到0的复合操作；这样一个复合操作用一个ReplicaSet是不太好描述的，所以用一个更通用的Deployment来描述。以K8s的发展方向，未来对所有长期伺服型的的业务的管理，都会通过Deployment来管理。

## StatefulSet
StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计），其应用场景包括:
* 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
* 稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现
* 有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现
* 有序收缩，有序删除（即从N-1到0）

## DaemonSet
长期伺服型和批处理型服务的核心在业务应用，可能有些节点运行多个同类业务的Pod，有些节点上又没有这类Pod运行；而后台支撑型服务的核心关注点在K8s集群中的节点（物理机或虚拟机），要保证每个节点上都有一个此类Pod运行。节点可能是所有集群节点也可能是通过nodeSelector选定的一些特定节点。典型的后台支撑型服务包括，存储，日志和监控等在每个节点上支持K8s集群运行的服务。

## PersistentVolume
PersistentVolumes提供存储的抽象概念，它记录复杂的存储细节（使用的存储驱动及相关配置信息），使得Pod挂载卷可以忽略这些繁杂的细节，可以把一个PersistentVolume看作一个抽象卷。

## PersistentVolumeClaim
存储的PV和PVC的这种关系，跟计算的Node和Pod的关系是非常类似的；PV和Node是资源的提供者，根据集群的基础设施变化而变化，由K8s集群管理员配置；而PVC和Pod是资源的使用者，根据业务服务的需求变化而变化，有K8s集群的使用者即服务的管理员来配置。

## Secret
Secret是用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。使用Secret的好处是可以避免把敏感信息明文写在配置文件里。

## Namespace
名字空间为K8s集群提供虚拟的隔离作用，K8s集群初始有两个名字空间，分别是默认名字空间default和系统名字空间kube-system，除此以外，管理员可以可以创建新的名字空间满足需要。

## UserAccount
用户帐户为人提供账户标识，记录了权限以及凭证信息。用户帐户对应的是人的身份，人的身份与服务的namespace无关，所以用户账户是跨namespace的。

## ServiceAccount
服务账户为计算机进程和K8s集群中运行的Pod提供账户标识，只记录了权限信息。服务帐户对应的是一个运行中程序的身份，与特定namespace是相关的。

# 核心组件
![K8s架构图](https://res.cloudinary.com/dukp6c7f7/image/upload/f_auto,fl_lossy,q_auto/s3-ghost/2016/06/o7leok.png)

## etcd
数据库。所有master的持续状态都存在etcd的一个实例中。这可以很好地存储配置数据。因为有watch(观察者)的支持，各部件协调中的改变可以很快被察觉。

## apiserver
作为K8s集群对外的开放API的门户入口，主要功能有：
* 提供整个K8s集群资源增删改查的Restful接口。
* 对接口请求进行安全校验。 
* 把资源数据持久化到etcd。
* 作为资源分配的入口点。scheduler监测到有容器需要创建时，经过sheduler调度算法计算好资源分配策略后需要调用apiserver的接口进行资源的实际分配操作，最终由apiserver向kubelet节点发起容器创建的通知。

## controller-manager
集群内部的Controller管理中心，内置了多个原生Controller。

说到controller-manager不得不介绍Controller组件，apiServer对外提供的Restful接口纯粹是对资源数据的增删改查操作，对于资源的部署和维护等逻辑不包含在其中，实际上是Controller通过apiServer监控集群的资源数据变化，发现某个资源数据发生变化时会做出相应的处理逻辑。

比如：controller-manager中的Replication Controller（不同于资源类型的ReplicationController，为了便于区分接下来我会把资源类型的ReplicationController简称为RC）会检测集群中的POD节点情况，确保RC中的POD数量保持在用户创建RC时指定的副本数，当发现POD宕机时，Replication Controller会重新创建一个新的POD来维持这个副本数；当用户修改了RC的副本数时，Replication Controller也会根据当前RC中POD的数据进行POD的追加或删除。

## scheduler
Scheduler负责Pod调度。负责接收Controller Manager创建的新的Pod，根据调度策略为其分配一个合适的Node。完整的调度过程为：
1. 通过调度算法为待调度Pod列表的每个Pod从Node列表中选择一个最适合的Node，并将信息写入etcd中。
2. kubelet通过API Server监听到kubernetes Scheduler产生的Pod绑定信息，然后获取对应的Pod清单，下载Image，并启动容器。

![scheduler调度过程](https://upload-images.jianshu.io/upload_images/7378149-d8078bd4c09c75a1.png?imageMogr2/auto-orient/strip|imageView2/2/w/603/format/webp)

## kubelet
kubelet充当K8s worker节点对容器生命周期的管理服务。经过上述scheduler调度策略完成之后，需要真正在对应的Node上创建这个POD，kubelet就是负责这一部分的工作。

kubelet是基于controller模式实现的（笔者日后会编写关于自定义Controller的文章讲解），它会监听Pod的增删改事件（包括Pod的生命周期变化）、Node自身事件（每个kubelet就是一个Node）以及定时的清理事件。一旦发现监听的数据产生了变化，kubelet会立刻作出相应的业务逻辑，如：scheduler把POD分配给了一个Node，这个Node监听到POD数据修改事件并从最新的POD数据中得知该POD被分配到当前Node中，kubelet判断到当前宿主机资源满足容器创建后就会根据POD的容器配置在宿主机上创建这些容器。

## kube-proxy
kube-proxy在K8s中主要承担路由转发和负载均衡的工作，具体功能包括：
* 管理service的访问入口，包括集群内Pod到Service的访问和集群外访问service。
* 管理sevice的Endpoints，该service对外暴露一个Virtual IP，也成为Cluster IP, 集群内通过访问这个Cluster IP:Port就能访问到集群内对应的serivce下的Pod。
* service是通过Selector选择的一组Pods的服务抽象，其实就是一个微服务，提供了服务的LB和反向代理的能力，而kube-proxy的主要作用就是负责service的实现。

## kubectl
apiserver开放的接口是HTTPS协议的Restful API，不方便用户在控制台直接操作K8s。kubectl就是为了解决这个问题而产生的，用户可以在控制台中通过kubectl封装好的简单命令对K8s集群进行操作（如：kubectl get pod 查询默认命名空间下的pod），kubectl会根据用户输入的命令转化成apiserver Restful API的请求形式再转发给apiserver。