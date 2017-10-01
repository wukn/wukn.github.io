---
title: "[k8s] 部署kubernetes集群——（二）部署etcd和flannel"
author: "wukn"
tags: ["k8s"]
catalog: true
date: 2017-10-02T00:00:00+08:00
draft: false
url: /2017/10/01/install-kubernetes-cluster-v1.8.0-deploy-master/

---

> Kubernetes使用etcd存储集群状态数据。因此需要部署一个etcd集群，这里复用master的三台机器。

> Kubernetes集群的Pod之前需要能相互通信，这里使用flannel搭建vxlan网络。

<!--more-->

### 部署etcd

三个机器分别取个etcd节点名称：

* 172.10.26.101: etcd-node1
* 172.10.26.102: etcd-node2
* 172.10.26.103: etcd-node3

#### 相关变量

```shell
# etcd节点名称（etcd集群内唯一）
NODE_NAME=etcd-node1

# 当前部署的机器IP
NODE_IP=172.10.26.101

# etcd集群间通信的IP和端口
ETCD_NODES=etcd-node1=https://172.10.26.101:2380,etcd-node2=https://172.10.26.102:2380,etcd-node3=https://172.10.26.103:2380
```

#### 下载etcd

```shell
curl -LO https://github.com/coreos/etcd/releases/download/v3.2.8/etcd-v3.2.8-linux-amd64.tar.gz
tar xvf etcd-v3.2.8-linux-amd64.tar.gz
mv etcd-v3.2.8-linux-amd64/etcd* /usr/local/bin
```

#### 创建tls证书和密钥

创建etcd证书签名请求：

```bash
cat > etcd-csr.json <<EOF 
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}"
  ],
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
```

生成证书和密钥：

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
mkdir -p /etc/etcd/ssl
mv etcd*.pem /etc/etcd/ssl
```

#### 创建etcd的服务文件

```shell
# 创建工作目录
mkdir -p /var/lib/etcd

cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \\
  --name=${NODE_NAME} \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\
  --listen-peer-urls=https://${NODE_IP}:2380 \\
  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://${NODE_IP}:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

#### 启动etcd服务

```shell
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

查看集群健康状态：

```shell
IPS="172.10.26.101 172.10.26.102 172.10.26.103"
for ip in $IPS; do
    ETCDCTL_API=3 etcdctl --endpoints=https://${ip}:2379  \
        --cacert=/etc/kubernetes/ssl/ca.pem \
        --cert=/etc/etcd/ssl/etcd.pem \
        --key=/etc/etcd/ssl/etcd-key.pem \
        endpoint health
done
```

### 配置flannel网络

Kubernetes内的个服务以及Pod之间的需要能相互访问。一般使用flannel搭建vxlan网络。master节点和node节点上都需要部署flannel。

#### 相关变量

```shell
# etcd集群服务地址列表
ETCD_ENDPOINTS="https://172.10.26.101:2379,https://172.10.26.102:2379,https://172.10.26.103:2379"

# POD网段（Cluster CIDR）
CLUSTER_CIDR="172.11.0.0/16"

# 服务端口范围（NodePort Range）
NODE_PORT_RANGE="8400-9000"

# flannel网络配置前缀
FLANNEL_ETCD_PREFIX="/kubernetes/network"
```

#### 创建tls证书和密钥

etcd集群启用了双向tls认证，因此我们需要为flannel创建与etcd集群通信的证书和密钥。

创建flannel证书签名请求：

```shell
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

生成证书和密钥：

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld

mkdir -p /etc/flanneld/ssl
mv flanneld*.pem /etc/flanneld/ssl
```

#### 在etcd中配置集群Pod网段

该操作只需要在执行一次即可。

```shell
etcdctl \
    --endpoints=${ETCD_ENDPOINTS} \
    --ca-file=/etc/kubernetes/ssl/ca.pem \
    --cert-file=/etc/flanneld/ssl/flanneld.pem \
    --key-file=/etc/flanneld/ssl/flanneld-key.pem \
    set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
```

注意，参数`CLUSTER_CIDR`应该与kube-controller-manager的--cluster-cidr值一致。

#### 下载flannel

```shell
curl -LO https://github.com/coreos/flannel/releases/download/v0.9.0/flannel-v0.9.0-linux-amd64.tar.gz
tar xvf flannel-v0.9.0-linux-amd64.tar.gz

mv {flanneld,mk-docker-opts.sh} /usr/local/bin
```

#### 创建flannel的服务文件

```shell
cat > /etc/systemd/system/flanneld.service << EOF
[Unit]
Description=Flanneld service
After=network.target
After=network-online.target
Wants=network-online.target
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \\
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  -etcd-certfile=/etc/flanneld/ssl/flanneld.pem \\
  -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem \\
  -etcd-endpoints=${ETCD_ENDPOINTS} \\
  -etcd-prefix=${FLANNEL_ETCD_PREFIX}
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```

#### 启动flanneld服务

```shell
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```

查看集群Pod网段：

```shell
etcdctl \
    --endpoints=${ETCD_ENDPOINTS} \
    --ca-file=/etc/kubernetes/ssl/ca.pem \
    --cert-file=/etc/flanneld/ssl/flanneld.pem \
    --key-file=/etc/flanneld/ssl/flanneld-key.pem \
    get ${FLANNEL_ETCD_PREFIX}/config
```

查看已分配的Pod子网段列表：

```shell
etcdctl \
    --endpoints=${ETCD_ENDPOINTS} \
    --ca-file=/etc/kubernetes/ssl/ca.pem \
    --cert-file=/etc/flanneld/ssl/flanneld.pem \
    --key-file=/etc/flanneld/ssl/flanneld-key.pem \
    get ${FLANNEL_ETCD_PREFIX}/config
```

查看某一Pod网段的相关参数：

```shell
etcdctl \
    --endpoints=${ETCD_ENDPOINTS} \
    --ca-file=/etc/kubernetes/ssl/ca.pem \
    --cert-file=/etc/flanneld/ssl/flanneld.pem \
    --key-file=/etc/flanneld/ssl/flanneld-key.pem \
    get ${FLANNEL_ETCD_PREFIX}/subnets/172.11.95.0-24
```

---

参考资料：

https://github.com/opsnull/follow-me-install-kubernetes-cluster

