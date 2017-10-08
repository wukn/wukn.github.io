---
title: "[k8s] 部署kubernetes集群——（一）准备工作"
author: "wukn"
tags: ["k8s"]
catalog: true
date: 2017-10-01T15:00:00+08:00
draft: false
url: /2017/10/01/install-kubernetes-cluster-v1.8.0-prepare/

---

> Kubernetes集群节点分为master和node角色，master节点上需要kube-apiserver、kube-scheduler、kube-controller-manager。node节点上需要flannel、docker、kubelet、kube-proxy。master的高可用依赖etcd。集群内容器通信使用flannel搭建的vxlan网络。组件之间的通信使用TLS证书进行加密。使用时通常还需要kubedns、dashboard、heapster、docker registry等组组件。

<!--more-->

### 集群机器

OS：CentOS 7.3.1611

kubernetes master、etcd：

* 172.10.26.101
* 172.10.26.102
* 172.10.26.103

kubernetes node：

* 172.10.26.111
* 172.10.26.112

Kubernetes系统的各个组件需要使用TLS证书对通信进行加密。我们可以使用CloudFlare的cfssl来生成CA证书和密钥。CA是自签名证书，用来签名后续创建的其它TLS证书。

### 创建CA证书和密钥

#### 安装cfssl

下载cfss、cfssljson和cfssl-certinfo，生成默认的配置文件和csr文件：

```shell
curl -O https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

curl -O https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

curl -O https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

export PATH=/usr/local/bin:$PATH
mkdir -p ssl
cd ssl
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
```

### 创建CA证书

创建CA配置文件`ca-config.json`：

```shell
cat << EOF > ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}
EOF

ls -l ca-config.json
```

证书中可以定义多个profile，分别指定不同的使用场景和过期时间。后续在给证书签名时使用某个profile。

* `signing`表示该证书可以用于签名其它证书
* `server auth`表示client可以用该CA对server提供的证书进行验证
* `client auth`表示server可以用该CA对client提供的证书进行验证

创建CA证书签名请求`ca-csr.json`：

```shell
cat << EOF > ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "NanJing",
      "ST": "NanJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

ls -l ca-csr.json
```

* `CN`即Common Name，kube-apiserver从证书中提取该字段作为请求的UserName；浏览器使用该字段验证网站是否合法
* `O`即Organization，kube-apiserver从证书中提取该字段作为请求用户所属的Group

生成证书和密钥：

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

将证书分发到所有机器的`/etc/kubernetes/ssl`目录下：

```shell
mkdir -p /etc/kubernetes/ssl
cp ca* /etc/kubernetes/ssl

scp /etc/kubernetes/ssl/* 172.10.26.102:/etc/kubernetes/ssl/
scp /etc/kubernetes/ssl/* 172.10.26.103:/etc/kubernetes/ssl/
scp /etc/kubernetes/ssl/ca.pem 172.10.26.111:/etc/kubernetes/ssl
scp /etc/kubernetes/ssl/ca.pem 172.10.26.112:/etc/kubernetes/ssl
```

### 安装集群管理工具kubectl

kubectl是管理kubernetes集群的命令行工具。我们在三个master节点都都安装一下。

#### 下载kubectl

```shell
curl -LO https://dl.k8s.io/v1.8.0/kubernetes-client-linux-amd64.tar.gz
tar xvf kubernetes-client-linux-amd64.tar.gz
cp kubernetes/client/bin/kube* /usr/local/bin/
chmod a+x /usr/local/bin/kube*
```

#### 创建admin证书

kubectl与kube-apiserver通信需要tls证书确保安全通信。

在上面创建的ssl目录下创建admin证书签名请求`admin-csr.json`：

```shell
cat << EOF > admin-csr.json
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
      "L": "NanJing",
      "ST": "NanJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

ls -l admin-csr.json
```

* 后续kube-apiserver使用RBAC对客户端（如kubelet、kube-proxy、Pod）请求进行授权
* kube-apiserver预定义了一些RBAC使用的RoleBindings，如cluster-admin将Group`system:masters`与Role`cluster-admin`绑定，该Role授予了调用kube-apiserver所有API的权限
* `O`指定该证书的Group为system:masters，kubelet使用该证书访问kube-apiserver时，由于证书被CA签名，所以认证通过，同时由于证书用户组为经过预授权的system:masters，所以被授予访问所有API的权限
* `hosts`为空列表

生成admin证书和密钥：

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin

mv admin*.pem /etc/kubernetes/ssl/

scp /etc/kubernetes/ssl/admin* 172.10.26.102:/etc/kubernetes/ssl/
scp /etc/kubernetes/ssl/admin* 172.10.26.103:/etc/kubernetes/ssl/
```

#### 创建kubectl配置文件

kubectl的默认配置文件路径为`~/.kube/config`，其中包含kube-apiserver地址、证书、用户名等信息。

```shell
# 注意将IP地址改成当前集群中任一master的IP
KUBE_APISERVER="https://172.10.26.103:6443"

# 设置集群参数
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER}

# 设置客户端认证参数
kubectl config set-credentials admin \
    --client-certificate=/etc/kubernetes/ssl/admin.pem \
    --embed-certs=true \
    --client-key=/etc/kubernetes/ssl/admin-key.pem

# 设置上下文参数
kubectl config set-context kubernetes \
    --cluster=kubernetes \
    --user=admin

# 设置默认上下文
kubectl config use-context kubernetes
```

---

参考资料：

https://github.com/opsnull/follow-me-install-kubernetes-cluster

