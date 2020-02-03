---
title: "k8s kubeconfig"
category: k8s
---
想要向k8s集群发送指令都是通过kubectl程序进行，kubectl是Go语言编写的应用程序，其功能只是调用k8s集群的API与k8s集群进行通信。kubectl相当于k8s集群的客户端，所以它需要知道服务端的地址、连接证书等信息，kubectl就是通过读取kubeconfig文件获知这些配置。kubectl默认会从$HOME/.kube目录下查找文件名为config的文件，也可以通过设置环境变量$KUBECONFIG或者通过设置--kubeconfig去指定其它kubeconfig文件。

<!--more-->

下面给出配置文件的例子: 
```yaml
current-context: federal-context
apiVersion: v1
clusters:
- cluster:
    api-version: v1
    server: http://cow.org:8080
  name: cow-cluster
- cluster:
    certificate-authority: path/to/my/cafile
    server: https://horse.org:4443
  name: horse-cluster
- cluster:
    insecure-skip-tls-verify: true
    server: https://pig.org:443
  name: pig-cluster
contexts:
- context:
    cluster: horse-cluster
    namespace: chisel-ns
    user: green-user
  name: federal-context
- context:
    cluster: pig-cluster
    namespace: saw-ns
    user: black-user
  name: queen-anne-context
kind: Config
preferences:
  colors: true
users:
- name: blue-user
  user:
    token: blue-token
- name: green-user
  user:
    client-certificate: path/to/my/client/cert
    client-key: path/to/my/client/key
```

## cluster
```yaml
clusters:
- cluster:
    certificate-authority: path/to/my/cafile
    server: https://horse.org:4443
  name: horse-cluster
- cluster:
    insecure-skip-tls-verify: true
    server: https://pig.org:443
  name: pig-cluster
```
配置中 __clusters__ 节点是一个列表，列表中的元素是Map结构，存储了每个集群的信息。__clusters.cluster.server__ 用于配置k8s集群apiserver完整路径，一般采用TLS协议与apiserver进行通信就需要证书验证，__clusters.cluster.certificate-authority__ 用于指定TLS协议的证书路径，若apiserver配置了不需要证书校验这个环境，则通过配置 __clusters.cluster.insecure-skip-tls-verify: true__ 来跳过TLS协议的证书验证。因为可以配置多个集群的信息，我们就需要通过名称来指定访问哪个集群，所以还需要通过 __clusters.name__ 为每个集群命名。

除了读取配置文件之外，还可以通过 __kubectl config set-cluster__ 指令动态配置集群信息，指令定义如下（详情参考[官方文档](http://kubernetes.kansea.com/docs/user-guide/kubectl/kubectl_config_set-cluster/)）：
```sh
kubectl config set-cluster NAME [--server=server] [--certificate-authority=path/to/certficate/authority] [--insecure-skip-tls-verify=true]
```

## user
```yaml
users:
- name: blue-user
  user:
    token: blue-token
- name: green-user
  user:
    client-certificate: path/to/my/client/cert
    client-key: path/to/my/client/key
```
配置中 __users__ 节点是一个列表，列表中的元素是Map结构，存储了每个用户的认证信息。访问k8s集群时都会带上当前用户信息让服务端进行ACL鉴权，每个用户都需要通过 __name__ 节点配置名称。此外，根据服务端的认证方式的不同，通过 __client-certificate__、__client-key__、__token__ 或 __username/password__ 配置用户认证信息。

除了修改配置文件之外，还可以通过 __kubectl config set-credentials__ 指令动态配置用户认证信息，指令定义如下（详情参考[官方文档](http://kubernetes.kansea.com/docs/user-guide/kubectl/kubectl_config_set-credentials)）：
```sh
kubectl config set-credentials NAME [--client-certificate=path/to/certfile] [--client-key=path/to/keyfile] [--token=bearer_token] [--username=basic_user] [--password=basic_password]
```

## context
```yaml
contexts:
- context:
    cluster: horse-cluster
    namespace: chisel-ns
    user: green-user
  name: federal-context
```

正在编写

## current-context
```yaml
current-context: federal-context
```

正在编写
