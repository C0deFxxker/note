---
title: "Docker privileged"
category: docker
---
Docker的--privileged参数大约在0.6版被引入，使用该参数，container内的root拥有真正的root权限，否则，container内的root只是外部的一个普通用户权限。

<!--more-->

```sh
$ docker help run 
...
--privileged=false         Give extended privileges to this container
...
```

privileged启动的容器，可以看到很多host上的设备，并且可以执行mount，甚至允许你在docker容器中启动docker容器。

## Rancher简单部署K8s
Rancher的自动部署K8s集群正是使用这种机制来实现，宿主机只需要下载rancher-agent镜像并给出正确的参数来启动这个镜像的容器，rancher-agent容器中将会自动下载Rancher K8s集群需要用到的镜像、构建好网络组件、K8s组件容器等一系列操作，整个过程非常简单，一键完成。

```sh
$ sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.2.8 --server https://192.168.1.1 --token kxbtbs64wwp59zfnc49bdlvhn7x5nbggrl4t9vfnjnhd29b6zdp7jb --ca-checksum 54dc4ecdbcd8b5099693df4a1e4fbc91aca8e41f2cde2431a4a6cc7cddca8524 --etcd --controlplane --worker
efee4938be7173b59fa242e1bba48879cc03ee17f73567a13fc0fac2c5fa36a9
```