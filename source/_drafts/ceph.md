---
title: "Ceph简介"
category: Ceph
---
# Ceph存储类型
Ceph 提供三种存储类型：FS, RBD, Object。三种存储类型各有优劣，满足不同场景的使用。

## Ceph FS
这种类型与NFS使用方式一致，实现了文件系统抽象接口的存储方式，在linux中看到CephFS以挂载盘形式存在，可以直接以操作文件目录形式操作CephFS的挂载盘。实现成本低，随便一台机器设定一个目录为CephFS服务盘即可。但因为需要经过文件系统抽象接口的调用，相对于Ceph的其它两种存储类型的传输效率较慢。

## Ceph RBD
RBD全称RADOS block device，以设备形式呈现在linux系统中，通过直接调用底层磁盘设备方式对RBD存取，免去了文件系统接口的调用环节，效率是三种Ceph存储类型中最高的，但必须建立在裸盘或磁盘阵列上。

## Ceph Object
有原生的API，而且也兼容Swift和S3的API，Ceph最底层的存储单元是Object对象，每个Object包含元数据和原始数据。

# Ceph核心概念
## Monitor
一个Ceph集群需要多个Monitor组成的小集群，它们通过Paxos同步数据，用来保存OSD的元数据。

## OSD
Object Storage Device，负责响应客户端请求返回具体数据的进程。一个Ceph集群一般都有很多个OSD。

## MDS
Ceph Metadata Server，是CephFS服务依赖的元数据服务。

## PG
Placement Grouops，是一个逻辑的概念，一个PG包含多个OSD。引入PG这一层其实是为了更好的分配数据和定位数据。

## RADOS
Reliable Autonomic Distributed Object Store，为用户实现数据分配、Failover等集群操作。

## Librados
Librados是Rados提供库，因为RADOS是协议很难直接访问，因此上层的RBD、RGW和CephFS都是通过librados访问的，目前提供PHP、Ruby、Java、Python、C和C++支持。

## CRUSH
Ceph使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方。

## RGW
RADOS gateway，是Ceph对外提供的对象存储服务，接口与S3和Swift兼容。

# Ceph IO流程
![Ceph IO流程图](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/icNyEYk3VqGk91oZGzW0jMNv73lKibM81Q3Vk69XEF5k7AMHI00TtSyj0KcTL0uibaiczgw4z1gAYSxprypZGTVrmQ/640?wx_fmt=png)

使用CephFS上传文件时：
1. 会把文件切分成若干小块（默认4M一块）
2. 将切分的块序号与文件ID的对应关系记录在FS的元数据服务中
3. 根据文件ID与块序号计算ObjectID
4. 计算PGID=Hash(ObjectID) & mask，其中Hash函数是Ceph设置好的，mask是PG的总数，这个总数是2的整数幂-1。
5. 采用CRUSH算法将PGID作为参数得到一组OSD。OSD就是真正存储数据的集群服务。

PG是Placement Group的简称，它充当桶的作用，一个桶下可以有多个Object，这个桶是虚拟划分的，底层存储组件的数目可能不等于PG的数目，在创建池的时候设定PG数目。

# Ceph ObjectStorage
## Pool
管理物理结构（匹配哪些OSD）。

## Bucket
管理逻辑结构，在已有的Pool中管理抽象的对象数据。

# 会纪
1. 单个Ceph RGW Bucket对应的文件数达到10w时索引文件开始切分，期间无法继续写入，耗时较长。
2. 