---
title: "[k8s] 部署kubernetes集群——（六）部署Traefik"
author: "wukn"
tags: ["k8s"]
catalog: true
date: 2017-10-18T00:00:00+08:00
draft: false
url: /2017/10/18/install-kubernetes-cluster-v1.8.0-deploy-traefik/

---

> Traefik是一款开源的反向代理与负载均衡工具。它可以与常见的微服务系统直接整合，可以实现自动化动态配置。它原生支持Kubernetes作为后端。

<!--more-->

### 服务发现与负载均衡

Kubernetes中的负载均衡可以有以下集中方式：

* Service：使用Service提供集群内部的负载均衡，并借助云提供商提供的LB提供外部访问
* Ingress Controller：使用Service提供集群内部的负载均衡，但是通过自定义LB提供外部访问
* Service Load Balancer：把load balancer直接跑在容器中
* Custom Load Balancer：自定义负载均衡，并替代kube-proxy

#### Service

Service是对一组提供相同功能的Pods的抽象，并为它们提供一个统一的入口。借助Service，应用可以方便的实现服务发现与负载均衡，并实现应用的不停服升级。Service通过标签来选取服务后端，一般配合Replication Controller或者Deployment来保证后端容器的正常运行。

Service有ClusterIP、NodePort、LoadBalancer三种类型：

* ClusterIP为默认类型，仅分配一个集群内可以访问的虚拟IP。
* NodePort在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过`<NodeIP>:NodePort`来访问服务
* LoadBalancer在NodePort的基础上，借助云提供商创建一个外部的负载均衡器，将请求转发到`<NodeIP>:NodePort`

外部服务也可以使用Endpoint将服务加入集群中。

#### Ingress Controller

Service在对外访问的时候，NodePort类型需要在外部搭建额外的负载均衡，而LoadBalancer要求kubernetes必须跑在支持的云上面。为了解决这个限制，Kubernetes引入了Ingress，用来将服务暴露到集群外面，并且可以自定义服务的访问策略。

Ingress本身并不会自动创建负载均衡器，需要结合Ingress Controller使用。Traefik就是一个易用的Ingress Controller。

#### Service Load Balancer

在Ingress出现以前，Service Load Balancer是推荐的解决Service局限性的方式。Service Load Balancer将haproxy跑在容器中，并监控service和endpoint的变化，通过容器IP对外提供4层和7层负载均衡服务。

#### Custom Load Balancer

在一些复杂的场景下，例如需要接入已有的负载均衡设备，或者多租户网络情况下容器网络和主机网络是隔离的导致kube-proxy就不能正常工作，需要自定义组件，并代替kube-proxy来做负载均衡。基本思路是监控Kubernetes中service和endpoints的变化，根据这些变化来配置负载均衡器。比如weave flux、nginx plus、kube2haproxy等。

### 部署Traefik

#### 创建ingress-rbac.yaml

用于service account验证。

```shell
cat > ingress-rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: ingress
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

#### 创建ingress

为前面部署的my-nginx服务创建ingress，指定域名、URL路径、后端服务和端口：

```shell
cat > my-nginx-ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-ingress
  namespace: default
spec:
  rules:
  - host: my-nginx.wukn.io
    http:
      paths:
      - path: /
        backend:
          serviceName: my-nginx
          servicePort: 80
EOF
```

`backend`中的service默认是`default`命名空间下的。如果是其他命名空间的服务，需要在`backend`下配置`namespace`。

#### 创建Traefik的Deployment

```shell
cat > traefik-deployment.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: traefik-ingress-lb
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      restartPolicy: Always
      serviceAccountName: ingress
      containers:
      - image: traefik
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8580
          hostPort: 8580
        args:
        - --web
        - --web.address=:8580
        - --kubernetes
EOF
```

Traefik占用主机的80端口提供反向代理服务。这里没有限定该pod运行在哪个主机上，实际使用时可以在指定的几台主机上运行，实现高可用。Traefik的UI服务端口是8580。

#### 创建Traefik UI的ingress

```shell
cat > traefik-ui.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8580
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik.wukn.io
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
EOF
```

#### 部署traefik

```shell
kubectl create -f .
```

#### 验证traefik功能

查看traefik所在的Node：

```shell
kubectl --all-namespaces -o wide get pod | grep traefik

# output：
# kube-system   traefik-ingress-lb-779cccbd8c-tfn8c    1/1       Running   0          3m        172.10.26.112   172.10.26.112
```

访问`http://172.10.26.112:8580`打开Traefik的Dashboard。

使用curl测试通过traefik访问服务：

```shell
curl -H Host:my-nginx.wukn.io 172.10.26.112
```

或者在本机hosts文件里添加域名映射：

```shell
172.10.26.112 my-nginx.wukn.io
172.10.26.112 traefik.wukn.io
```

在浏览器中通过域名可以访问相应的服务。

---

参考资料：

https://jimmysong.io/kubernetes-handbook/practice/service-discovery-and-loadbalancing.html

https://jimmysong.io/kubernetes-handbook/practice/traefik-ingress-installation.html
