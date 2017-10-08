---
title: "[k8s] 部署kubernetes集群——（三）部署Node节点"
author: "wukn"
tags: ["k8s"]
catalog: true
date: 2017-10-08T00:00:00+08:00
draft: false
url: /2017/10/08/install-kubernetes-cluster-v1.8.0-deploy-node/

---

> Kubernetes node节点上需要docker、flannel、kublet和kube-proxy。

flannel的部署参见上上篇。

<!--more-->

在172.10.26.111和172.10.26.112上部署node相关服务。

### 相关变量

```shell
# 当前node节点的IP
NODE_IP=172.10.26.111

# 服务网段
SERVICE_CIDR="10.254.0.0/16"

# 集群DNS服务IP(从SERVICE_CIDR中预分配)
CLUSTER_DNS_SVC_IP="10.254.0.2"

# 集群DNS域名
CLUSTER_DNS_DOMAIN="cluster.local."
```

### 配置docker

#### 下载docker

```shell
curl -LO https://download.docker.com/linux/static/stable/x86_64/docker-17.09.0-ce.tgz
tar xvf docker-17.09.0-ce.tgz
cp docker/docker* /usr/local/bin
```

#### 创建docker的服务文件

```shell
cat  > /etc/systemd/system/docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
Environment="PATH=/usr/local/bin:/bin:/sbin:/usr/bin:/usr/sbin"
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/local/bin/dockerd --log-level=error \$DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

* flanneld启动时将网络配置写入到/run/flannel/docker文件中的变量DOCKER_NETWORK_OPTIONS，docker使用该参数设置docker0网桥
* 默认开启的`--iptables`和`--ip-masq`不能关闭
* 设置iptable FORWARD chain策略为true：`iptables -P FORWARD ACCEPT`

#### 启动docker服务

```shell
systemctl daemon-reload
systemctl stop firewalld
systemctl disable firewalld
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
systemctl enable docker
systemctl start docker
systemctl status docker
```

这里禁用了firewalld，避免重复创建iptables规则；同时清理了旧的iptables规则。

使用`docker version`或`docker info`命令检查的docker服务是否成功启动。

### 配置kubelet

kubelet启动时向kube-apiserver发送TLS bootstrapping请求，需要先将bootstrap token文件中的kubelet-bootstrap用户赋予`system:node-bootstrapper`角色，然后kubelet才有权限创建认证请求。

在配置了kubectl的节点上执行：

```shell
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

`--user=kubelet-bootstrap`是/etc/kubernetes/token.csv中指定的用户名，同时也写入了/etc/kubernetes/bootstrap.kubeconfig。

#### 下载kubelet和kube-proxy

从server节点上拷贝kubelet和kube-proxy二进制文件到node节点上：

```shell
# 可以下载二进制文件
# curl -LO https://dl.k8s.io/v1.8.0/kubernetes-node-linux-amd64.tar.gz

# 也可以从server节点拷贝
scp /usr/local/bin/{kube-proxy,kubelet} 172.10.26.111:/usr/local/bin
scp /usr/local/bin/{kube-proxy,kubelet} 172.10.26.112:/usr/local/bin
```

#### 创建kubelet的bootstrapping.kubeconfig文件

在配置了kubectl的节点上执行：

```shell
# 相关变量
KUBE_APISERVER="https://172.10.26.101:6443" # master集群中任一节点的IP
BOOTSTRAP_TOKEN="7033223b0a7bf1f302654f1e86946939" # 上一篇中生成的token

# 设置集群参数
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
    --token=${BOOTSTRAP_TOKEN} \
    --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
    --cluster=kubernetes \
    --user=kubelet-bootstrap \
    --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

scp bootstrap.kubeconfig 172.10.26.111:/etc/kubernetes/
scp bootstrap.kubeconfig 172.10.26.112:/etc/kubernetes/
```

这里设置kubelet客户端认证参数时没有指定秘钥和证书，后续由kube-apiserver自动生成。

#### 创建kubelet的服务文件

```shell
# 创建工作目录
mkdir /var/lib/kubelet

cat > /etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \\
  --address=${NODE_IP} \\
  --hostname-override=${NODE_IP} \\
  --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest \\
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --require-kubeconfig \\
  --cert-dir=/etc/kubernetes/ssl \\
  --cluster-dns=${CLUSTER_DNS_SVC_IP} \\
  --cluster-domain=${CLUSTER_DNS_DOMAIN} \\
  --hairpin-mode promiscuous-bridge \\
  --allow-privileged=true \\
  --serialize-image-pulls=false \\
  --logtostderr=true \\
  --v=2
ExecStartPost=/sbin/iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -s 172.16.0.0/12 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -s 192.168.0.0/16 -p tcp --dport 4194 -j ACCEPT
ExecStartPost=/sbin/iptables -A INPUT -p tcp --dport 4194 -j DROP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

* `--address`不能设置为127.0.0.1，否则后续Pods访问kubelet的API接口时会失败，因为Pods访问的127.0.0.1指向自己而不是kubelet
* 如果设置了`--hostname-override`，则kube-proxy也要设置该选项，否则会出现找不到Node的情况
* `--experimental-bootstrap-kubeconfig`指向bootstrap.kubeconfig文件，kubelet使用其中的用户名和token向kube-apiserver发送TLS Bootstrapping请求
* 管理员通过了CSR请求后，kubelet会在`--cert-dir`目录创建证书kubelet-client.crt和密钥kubelet-client.key，然后写入`--kubeconfig`指定的文件
* `--cluster-dns`和`--cluster-domain`分别指定kubedns的Service IP和域名后缀
* kubelet cAdvisor默认监听4194端口，ExecStartPost通过iptables限定只有内网IP能够访问
* kubelet默认要求swap交换分区是关闭的，或者指定`--fail-swap-on flag=false`来忽略它

#### 启动kubelet服务

```shell
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

#### 通过kubelet的TLS证书请求

kubelet首次启动时向kube-apiserver发送证书签名请求，必须通过后kubernetes才会将该Node加入到集群。

```shell
# 查看未授权的CSR请求
kubectl get csr

# 结果大概是这样：
# NAME                                                   AGE       REQUESTOR           CONDITION
# node-csr-DNmVZKP5HOWM6kWdZrQXrB1T5b6oVuEj7zLafmeP7z4   5m        kubelet-bootstrap   Pending
# node-csr-j1zE8OAiNpQL15Z2sdaZkQMtnNKc50nXY4yLSFrLTF0   3m        kubelet-bootstrap   Pending

# 查看集群中的node
kubectl get nodes

# 通过CSR请求
kubectl certificate approve node-csr-DNmVZKP5HOWM6kWdZrQXrB1T5b6oVuEj7zLafmeP7z4
kubectl certificate approve node-csr-j1zE8OAiNpQL15Z2sdaZkQMtnNKc50nXY4yLSFrLTF0

# 再次查看未授权的CSR请求
kubectl get csr

# 结果变成了这样：
# NAME                                                   AGE       REQUESTOR           CONDITION
# node-csr-DNmVZKP5HOWM6kWdZrQXrB1T5b6oVuEj7zLafmeP7z4   6m        kubelet-bootstrap   Approved,Issued
# node-csr-j1zE8OAiNpQL15Z2sdaZkQMtnNKc50nXY4yLSFrLTF0   4m        kubelet-bootstrap   Approved,Issued

# 再次查看集群中的node
kubectl get nodes

# 现在集群中能看到了node节点了
# NAME            STATUS    ROLES     AGE       VERSION
# 172.10.26.111   Ready     <none>    13m       v1.8.0
# 172.10.26.112   Ready     <none>    13m       v1.8.0
```

看一下自动生成的kubelet kubeconfig文件和公私钥：

```shell
ls -l /etc/kubernetes/kubelet.kubeconfig
ls -l /etc/kubernetes/ssl/kubelet*
```

### 配置kube-proxy

#### 创建kube-proxy证书

创建kube-proxy证书签名请求：

```shell
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
      "L": "NanJing",
      "ST": "NanJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

* `CN`中指定User为`system:kube-proxy`
* kube-apiserver预定义的RoleBinding `system:node-proxier`将User `system:kube-proxy`与Role `system:node-proxier`绑定，该Role授予了调用kube-apiserver Proxy相关API的权限

生成kube-proxy客户端证书和私钥：

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
    -ca-key=/etc/kubernetes/ssl/ca-key.pem \
    -config=/etc/kubernetes/ssl/ca-config.json \
    -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
mv kube-proxy*.pem /etc/kubernetes/ssl/
```

#### 创建kube-proxy.kubeconfig配置文件

在配置了kubectl的节点上执行：

```shell
# 相关参数
KUBE_APISERVER="https://172.10.26.101:6443" # master集群中任一节点的IP

# 设置集群参数
kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=kube-proxy.kubeconfig

# 设置客户端认证参数，注意这里需要proxy的证书和私钥
kubectl config set-credentials kube-proxy \
    --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
    --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

# 设置上下文参数
kubectl config set-context default \
    --cluster=kubernetes \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

scp kube-proxy.kubeconfig 172.10.26.111:/etc/kubernetes/
scp kube-proxy.kubeconfig 172.10.26.112:/etc/kubernetes/
```

#### 创建kube-proxy的服务文件

```shell
# 创建工作目录
mkdir /var/lib/kube-proxy

cat > /etc/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \\
  --bind-address=${NODE_IP} \\
  --hostname-override=${NODE_IP} \\
  --cluster-cidr=${SERVICE_CIDR} \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

* `--hostname-override`必须与kubelet一致
* `--cluster-cidr`必须与kube-apiserver的`--service-cluster-ip-range`一致
* kube-proxy根据`--cluster-cidr`判断集群内部和外部流量，指定`--cluster-cidr`或`--masquerade-all`选项后kube-proxy才会对访问Service IP的请求做SNAT

#### 启动kube-proxy服务

```shell
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```

### 验证集群功能

我们部署一个简单的nginx服务来验证集群的功能。

创建文件：

```shell
cat > nginx-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80

---

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.13.5
        ports:
        - containerPort: 80
EOF
```

创建Pod和服务：

```shell
kubectl create -f nginx-ds.yml
```

检查Node状态，STATE为Ready即为正常：

```shell
kubectl get nodes

# output
# NAME            STATUS    ROLES     AGE       VERSION
# 172.10.26.111   Ready     <none>    56m       v1.8.0
# 172.10.26.112   Ready     <none>    56m       v1.8.0
```

查看pod的IP：

```shell
kubectl get pods  -o wide | grep nginx-ds

# output
# nginx-ds-kswkm   1/1       Running   0          3m        172.11.40.2   172.10.26.112
# nginx-ds-m2q4m   1/1       Running   0          3m        172.11.84.2   172.10.26.111
```

查看服务的IP和状态：

```shell
kubectl get svc | grep nginx-ds

# output
# nginx-ds     NodePort    10.254.1.166   <none>        80:8733/TCP   5m

# 服务IP：10.254.1.166
# 服务端口：80
# NodePort端口：8733

# 在所有Node上执行curl http://10.254.1.166应该会输出nginx默认的首页
```

查看NodePort的是否可达：

```
curl http://172.10.26.111:8733

curl http://172.10.26.112:8773
```

---

参考资料：

https://github.com/opsnull/follow-me-install-kubernetes-cluster
