---
title: "[k8s] 部署kubernetes集群——（五）部署DNS、Dashboard和Heapster插件"
author: "wukn"
tags: ["k8s"]
catalog: true
date: 2017-10-09T00:00:00+08:00
draft: false
url: /2017/10/09/install-kubernetes-cluster-v1.8.0-deploy-dns-dashboard-heapster/

---

> 在Kubernetes集群内部署DNS、Dashboard和Heapster插件。

<!--more-->

### 部署DNS

进入kubernetes源码的cluster/addons/dns目录。

```shell
cd cluster/addons/dns
```

用到的文件有`kubedns-cm.yaml`、`kubedns-sa.yaml`、`kubedns-controller.yaml.base`、`kubedns-svc.yaml.base`。

#### 系统内置的RoleBinding

内置的RoleBinding `system:kube-dns`将`kube-system`命名空间的`kube-dns` ServiceAccount与`system:kube-dns` Role绑定，该Role具有访问kube-apiserver DNS相关API的权限：

```shell
kubectl get clusterrolebindings system:kube-dns -o yaml

# output：
# apiVersion: rbac.authorization.k8s.io/v1
# kind: ClusterRoleBinding
# metadata:
# annotations:
#   rbac.authorization.kubernetes.io/autoupdate: "true"
#   creationTimestamp: 2017-10-08T03:40:37Z
#   labels:
#     kubernetes.io/bootstrapping: rbac-defaults
#   name: system:kube-dns
#   resourceVersion: "78"
#   selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Akube-dns
#   uid: 72e6895d-abda-11e7-a434-525400474652
# roleRef:
#   apiGroup: rbac.authorization.k8s.io
#   kind: ClusterRole
#   name: system:kube-dns
# subjects:
# - kind: ServiceAccount
#   name: kube-dns
#   namespace: kube-system
```

`kubedns-controller.yaml`中定义的Pods时使用了`kubedns-sa.yaml`文件定义的`kube-dns` ServiceAccount，所以具有访问kube-apiserver DNS相关API的权限。

#### 配置kube-dns Service

```shell
cp kubedns-svc.yaml.base kubedns-svc.yaml

sed -i 's/__PILLAR__DNS__SERVER__/10.254.0.2/' kubedns-svc.yaml
```

这里需要将`spec.clusterIP`设置为变量CLUSTER_DNS_SVC_IP的值。

#### 配置kube-dns Deployment

```shell
cp kubedns-controller.yaml.base kubedns-controller.yaml

sed -i 's/__PILLAR__DNS__DOMAIN__\./cluster.local./g' kubedns-controller.yaml
sed -i 's/__PILLAR__DNS__DOMAIN__/cluster.local./g' kubedns-controller.yaml
```

`--domain`设置为变量CLUSTER_DNS_DOMAIN的值。

#### 部署kube-dns

```shell
kubectl create -f .
```

#### 验证dns功能

新建一个Deployment：

```shell
cat > my-nginx.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.13.5
        ports:
        - containerPort: 80
EOF

kubectl create -f my-nginx.yaml
```

导出该Deployment, 生成`my-nginx`服务：

```shell
kubectl expose deploy my-nginx

kubectl get services --all-namespaces |grep my-nginx

# output：
# default       my-nginx     ClusterIP   10.254.103.138   <none>        80/TCP          1s
```

创建另一个Pod，查看`/etc/resolv.conf`是否包含kubelet配置的`--cluster-dns`和`--cluster-domain`，是否能够将服务`my-nginx`解析到上面显示的Cluster IP：

```shell
cat > pod-nginx.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
EOF

kubectl create -f pod-nginx.yaml
kubectl exec  nginx -i -t -- /bin/bash

# 进入pod之后，在容器内运行
cat /etc/resolv.conf

# output：
# nameserver 10.254.0.2
# search default.svc.cluster.local. svc.cluster.local. cluster.local.
# options ndots:5

# 可以看到“nameserver 10.254.0.2”

# ping服务名可以得到对应的IP地址：

ping my-nginx

# output：
# PING my-nginx.default.svc.cluster.local (10.254.103.138): 48 data bytes


ping kubernetes

# output：
# PING kubernetes.default.svc.cluster.local (10.254.0.1): 48 data bytes

ping kube-dns.kube-system.svc.cluster.local

# output：
# PING kube-dns.kube-system.svc.cluster.local (10.254.0.2): 48 data bytes

```

---

参考资料：

https://github.com/opsnull/follow-me-install-kubernetes-cluster
