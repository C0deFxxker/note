---
title: "k8s 自签证书"
category: k8s
---

K8s集群节点间通信以及开放接口都是采用TLS协议的，TLS协议需要用到CA签发证书，而权威机构的CA证书只能根据域名签署，内网IP调用的CA证书就需要自建来实现了。

<!--more-->

组件 | 需要使用的证书
---|---
etcd | ca.pem server.pem server-key.pem
flannel | ca.pem server.pem server-key.pem
kube-apiserver | ca.pem server.pem server-key.pem
kubelet | ca.pem ca-key.pem
kube-proxy | ca.pem kube-proxy.pem kube-proxy-key.pem
kubectl | ca.pem admin.pem admin-key.pem

# 安装证书生成证书工具cfssl
```sh
wget -O cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl && mv cfssl /usr/bin/
wget -O cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson && mv cfssljson /usr/bin/
```

# 生成X509证书
在存放证书的目录下执行下列脚本生成证书：

```sh
echo -e "\n开始生成CA自签证书: ca.pem, ca-key.pem, ca.csr"
cat > ca-config.json <<EOF
{
  "signing": {
    "default":{
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "expiry": "87600h",
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
EOF

cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

#-------------------

echo -e "\n开始生成server证书: server.pem, server-key.pem, server.csr"
cat > server-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "localhost",
    <<IPADDRS>>
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

sed -i "s/<<IPADDRS>>/$IP_ADDRS/g" server-csr.json

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

#----------------
echo -e "\n开始生成admin证书: admin.pem, admin-key.pem, admin.csr"
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

#---------
echo -e "\n开始生成kube-proxy证书: kube-proxy.pem, kube-proxy-key.pem, kube-proxy.csr"
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

执行完脚本后如果没有任何报错则会在当前目录下生成上述K8s需要的证书文件。
```sh
$ ls
admin-csr.json		ca-key.pem		client.pem		server-key.pem
admin-key.pem		ca.csr			kube-proxy-csr.json	server.csr
admin.csr		ca.pem			kube-proxy-key.pem	server.pem
admin.pem		client-csr.json		kube-proxy.csr
ca-config.json		client-key.pem		kube-proxy.pem
ca-csr.json		client.csr		server-csr.json
```