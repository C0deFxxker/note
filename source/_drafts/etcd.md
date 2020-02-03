---
title: "etcd安装及证书创建"
category: k8s
---
本章将介绍etcd的安装以及x509证书的创建，K8s的数据库存储使用的是etcd，为了保证集群K8s各组件都需要使用x509证书对通信进行加密和认证，所以etcd也需要创建x509证书再传给kube-apiserver使用。

<!--more-->

本文使用的etcd版本为__v3.4.2__。

# 安装etcd
etcd是Go语言编写的项目，安装方式十分简单，只需要下载源码然后执行源码目录中的 build 程序就能编译生成二进制文件到源码目录下的./bin目录。源码编译的机器环境必须要先装好Go语言环境，但运行etcd的服务器上并不需要装Go语言环境，下面直接给出Dockerfile：

```Dockerfile
FROM golang:1.12 as dev

RUN git clone https://github.com/etcd-io/etcd.git -b v3.4.2 /home/etcd
WORKDIR /home/etcd
RUN ./build

FROM ubuntu:18.04

COPY --from=dev /home/etcd/bin /usr/bin

WORKDIR /root

EXPOSE 2379
ENTRYPOINT ["etcd"]
```

# 生成证书
本文使用CloudFlare的PKI工具集cfssl创建证书。

## 安装cfssl
```sh
wget -O cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl && mv cfssl /usr/bin/
wget -O cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson && mv cfssljson /usr/bin/
```

## 创建CA根证书配置文件
新增 ca-config.json 文件，内容如下：
```json
{
    "signing": {
        "default": {
            "expiry": "8760h"
        },
        "profiles": {
            "server": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "8760h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

读者可以根据实际情况修改上述配置。
* profiles：指定了不同角色的配置信息。服务端使用server，客户端使用client，集群节点间认证使用peer。
* expiry：指定了证书的过期时间为8760小时（即1年）

## 创建CA根证书证书签名请求
新增 ca-csr.json 文件，内容如下：
```json
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "O": "Ctyun",
            "ST": "BeiJing",
            "OU": "ops"
        }
    ]
}
```

然后执行如下命令生成CA证书：
```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
执行完上述指令后就能在目录下看到新增的ca-key.pem（私钥）、ca.csr（证书签名请求）、ca.pem（证书）文件。

## 创建服务器和客户端证书
根据默认配置文件进行修改：cfssl print-defaults csr > server-csr.json
主要修改部分如下：
```json
"CN": "test.coreos.com",
"hosts": [
	"server.coreos.org",
	"*.coreos.org"
],
```

执行下面的命令生成服务端证书：
```sh
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server-csr.json | cfssljson -bare server
```
-profile指定了使用ca-config.json中的profile。  
执行完指令后会生成server.csr、server-key.pem、server.pem三个文件。

用同样的方法创建客户端证书即可。

# 运行etcd
创建完证书后，把证书文件同步到Docker容器中，并为etcd启动参数指定证书文件，命令如下：
```sh
docker run -v $PWD/cert:/root/cert -p2379:2379 --name etcd etcd:3.4.2 --cert-file=/root/cert/server.pem --key-file=/root/cert/server-key.pem --advertise-client-urls=https://0.0.0.0:2379 --listen-client-urls=https://0.0.0.0:2379
```

然后进行本地测试，确保证书认证功能是开启的。
```sh
$ docker exec -ti d2a2eca21ffe /bin/sh
# 客户端不使用证书进行存取操作得到报错信息
$ etcdctl get foo
{"level":"warn","ts":"2019-10-28T03:18:06.822Z","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-d8d946a9-af43-4824-8d87-16754c8a0c94/127.0.0.1:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest connection error: connection closed"}
Error: context deadline exceeded
# 客户端使用证书进行操作，响应成功
$ etcdctl --cacert="/root/cert/ca.pem" put foo abc
OK
$ etcdctl --cacert="/root/cert/ca.pem" get foo
abc
ok
```

因为开放的是HTTPS协议，还可以直接通过curl或postman进行测试：
```sh
$ curl --cacert cert/ca.crt https://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -v
```