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

### 部署Dashboard

#### 下载并部署

下载Dashboard v1.7.1部署文件并部署：

```shell
curl -LO https://raw.githubusercontent.com/kubernetes/dashboard/v1.7.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

给Service添加label：

```shell
sed -i '155i\ \ \ \ kubernetes.io\/cluster-service:\ "true"' kubernetes-dashboard.yaml
sed -i '156i\ \ \ \ addonmanager.kubernetes.io\/mode:\ Reconcile' kubernetes-dashboard.yaml
```

如果想通过NodePort方式访问Dashboard，则修改服务类型：

```shell
sed -i '158i\ \ type:\ NodePort' kubernetes-dashboard.yaml
```

最终的Service配置长这样：

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

部署Dashboard：

```shell
kubectl create -f kubernetes-dashboard.yaml
```

#### 访问Dashboard

访问Dashboard有三种方法：

（1）通过NodePort访问

上面指定了Service为NodePort类型，可以通过`http://NodeIP:nodePort`访问服务。

查询NodePort：

```shell
kubectl get services -n kube-system | grep dashboard

# output：
# kubernetes-dashboard   NodePort    10.254.237.250   <none>        443:8685/TCP    29m
```

在浏览器中访问`https://172.10.26.111:8685`，会跳转到登陆页面。

（2）通过kube-apiserver访问

获取集群服务地址列表：

```shell
kubectl cluster-info

# output：
# Kubernetes master is running at https://172.10.26.101:6443
# KubeDNS is running at https://172.10.26.101:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy
# kubernetes-dashboard is running at https://172.10.26.101:6443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
```

由于kube-apiserver开启了RBAC授权，而浏览器访问kube-apiserver的时候使用的是匿名证书，所以访问安全端口会导致授权失败。这里需要使用非安全端口访问kube-apiserver，同时改成https方式：`http://172.10.26.101:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy`。

（3）通过kubectl proxy访问

在部署了kubectl节点上启动代理：

```shell
kubectl proxy --address='172.10.26.101' --port=8086 --accept-hosts='^*$'
```

在浏览器中访问`http://172.10.26.101:8086/ui`，会跳转到`http://172.10.26.101:8086/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/`，这时访问不了，因为Dashboard服务使用的是https，改成`http://172.10.26.101:8086/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`就可以访问了。

#### 创建有集群管理员权限的ServiceAccount

部署文件中的kube-dashboard只被授予了很小的权限，这里我们重新创建一个ServiceAccount，绑定`cluster-admin`角色：

```shell
cat > kube-dashboard-admin.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
EOF

kubectl create -f kube-dashboard-admin.yaml
```

查看kubernetes-dashboard-admin对应的token：

```shell
kubectl get secrets --all-namespaces | grep kubernetes-dashboard-admin

# output：
# kube-system   kubernetes-dashboard-admin-token-gnxl2   kubernetes.io/service-account-token   3         45s

kubectl describe secret kubernetes-dashboard-admin-token-gnxl2 -n kube-system

# output：
# Name:         kubernetes-dashboard-admin-token-gnxl2
# Namespace:    kube-system
# Labels:       <none>
# Annotations:  kubernetes.io/service-account.name=kubernetes-dashboard-admin
#               kubernetes.io/service-account.uid=7fa85490-b01c-11e7-a3b6-525400474652
#
# Type:  kubernetes.io/service-account-token
#
# Data
# ====
# ca.crt:     1359 bytes
# namespace:  11 bytes
# token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1nbnhsMiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjdmYTg1NDkwLWIwMWMtMTFlNy1hM2I2LTUyNTQwMDQ3NDY1MiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.IFc5HO8I31EsZ2m3mAJGUn4f2uYMYY7xhxYDaM3vG6tv58cXx8or0pgN6n47-bjA17f5kPfhzXX93lvWvShNYqj1HGDZnnSRUDMgTt_P16mG55hO_SyxO2p4dNyWvavYuz--6NdLtt8BCsmwm_03tsifhWXFhVCGL4EBLQPozfJZJe_AfL2o7e73Quj1Ve0wpJWbvjCBUKQni7MEfH6JQJhEl14ulffE9WHzN26sh5LllKbFl6AMS_1Zf0zqDAcwLJcX9C4WXG6IZNeNI57resBhvoa0GK-XjCXmk8UpAxMWJ5QykEL4wJO0JRmk6RMpFlgmQiM7MrSUnXzUYFMwiA
```

使用该token登录即可。


### 部署Heapster

TODO

---

参考资料：

https://github.com/opsnull/follow-me-install-kubernetes-cluster
https://github.com/kubernetes/dashboard/wiki/Installation
