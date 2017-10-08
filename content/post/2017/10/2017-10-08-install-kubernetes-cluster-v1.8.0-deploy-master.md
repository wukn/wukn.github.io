---
title: "[k8s] 部署kubernetes集群——（三）部署Master节点"
author: "wukn"
tags: ["k8s"]
catalog: true
date: 2017-10-08T00:00:00+08:00
draft: false
url: /2017/10/08/install-kubernetes-cluster-v1.8.0-deploy-master/

---

> Kubernetes master节点上需要kube-apiserver、kube-scheduler、kube-controller-manager。

同一时刻只有一个kube-scheduler和kube-controller-manager实例处于工作状态。为了master的高可用，我们会在每个master节点上都部署一套kube-scheduler和kube-controller-manager，它们之间通过选举产生一个leader。

master节点与node节点上的Pods通过pod网络通信，因此需要在master节点上部署flannel网络。上一篇已经说了。

<!--more-->

### 相关变量

```shell
# 当前master节点的IP
MASTER_IP=172.10.26.101

# 服务网段
SERVICE_CIDR="10.254.0.0/16"

# POD网段
CLUSTER_CIDR="172.11.0.0/16"

# 服务端口范围(NodePort Range)
NODE_PORT_RANGE="8400-9000"

# etcd集群服务地址列表
ETCD_ENDPOINTS="https://172.10.26.101:2379,https://172.10.26.102:2379,https://172.10.26.103:2379"

# kubernetes服务IP(预分配，一般是SERVICE_CIDR中第一个IP)
CLUSTER_KUBERNETES_SVC_IP=10.254.0.1

# TLS Bootstrapping使用的Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
BOOTSTRAP_TOKEN="7033223b0a7bf1f302654f1e86946939"
```


### 下载master组件

```shell
curl -LO https://dl.k8s.io/v1.8.0/kubernetes-server-linux-amd64.tar.gz
tar xvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin
```

### 创建kubernetes证书

创建kubernetes证书签名请求：

```shell
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "${MASTER_IP}",
    "${CLUSTER_KUBERNETES_SVC_IP}",
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
      "L": "NanJing",
      "ST": "NanJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

证书签名请求里hosts指定了授权使用该证书的IP或域名，所以指定了master节点的IP；另外要添加kube-apiserver注册的名为kubernetes的服务IP(Service Cluster IP)，一般是kube-apiserver --service-cluster-ip-range选项值指定的网段的第一个IP，如"10.254.0.1"。

生成kubernetes证书和密钥：

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
mv kubernetes*.pem /etc/kubernetes/ssl/
```

### 配置kube-apiserver

#### 创建kube-apiserver使用的客户端token文件

kubelet首次启动时向kube-apiserver发送TLS Bootstrapping请求，kube-apiserver验证kubelet请求中的token是否与它配置的token.csv一致，如果一致则自动为kubelet生成证书和秘钥。

```shell
echo ${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap" > /etc/kubernetes/token.csv
```

#### 创建kube-apiserver的服务文件

```shell
cat  > /etc/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --advertise-address=${MASTER_IP} \\
  --bind-address=${MASTER_IP} \\
  --insecure-bind-address=${MASTER_IP} \\
  --authorization-mode=RBAC \\
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \\
  --kubelet-https=true \\
  --experimental-bootstrap-token-auth \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --service-node-port-range=${NODE_PORT_RANGE} \\
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \\
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --etcd-servers=${ETCD_ENDPOINTS} \\
  --enable-swagger-ui=true \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/lib/audit.log \\
  --event-ttl=1h \\
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

* kube-scheduler、kube-controller-manager一般和kube-apiserver部署在同一台机器上，它们使用非安全端口和kube-apiserver通信
* kubelet、kube-proxy、kubectl部署在其它Node节点上，如果通过安全端口访问kube-apiserver，则必须先通过TLS证书认证，再通过RBAC授权
* kube-proxy、kubectl通过证书里指定的User、Group实现RBAC授权
* kubelet TLS Boostrap机制，与--kubelet-certificate-authority、--kubelet-client-certificate和--kubelet-client-key选项互斥
* --admission-control值必须包含ServiceAccount，否则部署集群插件时会失败
* --service-cluster-ip-range指定Service Cluster IP地址段，该地址段不能路由可达
* --service-node-port-range指定NodePor的端口范围
* 默认kubernetes对象保存在etcd的/registry路径下，可通过--etcd-prefix参数进行配置

#### 启动kube-apiserver服务

```shell
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

### 配置kube-controller-manager

#### 创建kube-controller-manager的服务文件

```shell
cat > /etc/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=127.0.0.1 \\
  --master=http://${MASTER_IP}:8080 \\
  --allocate-node-cidrs=true \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --cluster-cidr=${CLUSTER_CIDR} \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

* --address必须为127.0.0.1，因为当前kube-apiserver期与scheduler、controller-manager在部署同一台机器
* --master指定使用非安全8080端口与kube-apiserver通信
* --cluster-cidr指定集群中Pod的CIDR范围，由flannel保证该网段在各Node间路由可达
* --service-cluster-ip-range指定集群中Service的CIDR范围，该网络在各Node间必须路由不可达，且必须和kube-apiserver中的配置一致
* --cluster-signing-* 指定的证书和私钥文件用来签名为TLS BootStrap创建的证书和密钥
* --root-ca-file用来对kube-apiserver证书进行校验，指定该参数后，才会在Pod容器的ServiceAccount中放置该CA证书文件
* --leader-elect=true指定Master集群选举产生一个提供服务的kube-controller-manager

#### 启动kube-controller-manager服务

```shell
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

### 配置kube-scheduler

#### 创建kube-scheduler的服务文件

```shell
cat > /etc/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --address=127.0.0.1 \\
  --master=http://${MASTER_IP}:8080 \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### 启动kube-scheduler服务

```shell
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

### 验证Master集群

```shell
kubectl get componentstatuses
```

结果如下：

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}
```

---

参考资料：

https://github.com/opsnull/follow-me-install-kubernetes-cluster
