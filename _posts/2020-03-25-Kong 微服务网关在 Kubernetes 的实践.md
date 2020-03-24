---
layout: post
title: Kong 微服务网关在 Kubernetes 的实践 - 转载
description: 本文主要介绍将 Kong 微服务网关作为 Kubernetes 集群统一入口的最佳实践。
tags: [Kubernetes]
---

本文主要介绍将 Kong 微服务网关作为 Kubernetes 集群统一入口的最佳实践，之前写过一篇文章使用 Nginx Ingress Controller 作为集群统一的流量入口：[使用 Kubernetes Ingress 对外暴露服务](https://qhh.me/2019/08/12/%E4%BD%BF%E7%94%A8-Kubernetes-Ingress-%E5%AF%B9%E5%A4%96%E6%9A%B4%E9%9C%B2%E6%9C%8D%E5%8A%A1/)，但是相比于 Kong Ingress Controller 来说，Kong 支持的功能更加强大，更适合微服务架构：

- 拥有庞大的插件生态，能轻易扩展 Kong 支持的功能，比如 API 认证，流控，访问限制等；
- Kong 服务本身和 Admin 管理 API 都集成在一个进程，通过端口区分两者，简化了部署的复杂度；
- Kong 节点的配置统一持久化到数据库，所有节点通过数据库共享数据，在 Ingress 更新后能实时同步到各个节点，而 Nginx Ingress Controller 是通过重新加载机制响应 Ingress 更新，这种方式代价比较大，可能会导致服务的短暂中断；
- Kong 有成熟的第三方管理 UI 和 Admin 管理 API 对接，从而能可视化管理 Kong 配置；

本文先介绍 Kong 微服务网关在 Kubernetes 的架构，然后进行架构实践，涉及到的话题有：

- Kong 微服务网关在 Kubernetes 的架构；
- helm 部署 Kong（包含 Kong Ingress Controller）；
- 部署 Konga；



## Kong 微服务网关在 Kubernetes 的架构

Kubernetes 简化了微服务架构，以 Service 为单位，代表一个个微服务，但是 k8s 集群整个网络对外是隔离的，在微服务架构下一般需要一个网关作为所有 API 的入口，在 k8s 中架构微服务同样也需要一个网关，作为集群统一的入口，作为服务消费方和提供方的交互中间件。Kong 可以充当这一网关角色，为集群提供统一的外部流量入口，集群内部 Service 之间通信通过 Service 名称调用：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124312526.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

那么 Kong 是如何在 k8s 集群上跑起来？具体机制是什么样的？
Kong 作为服务接入层，不仅提供了外部流量的接收转发，而且其本身还提供了 Admin 管理 API，通过 Admin 管理 API 实现 Kong 的路由转发等相关配置，这两项功能都是在一个进程中实现。

在 k8s 中 Kong 以 Pod 形式作为节点运行，Pod 通过 Deployment 或者 DaemenSet 管理。所有 Kong 节点共享一个数据库，因此通过 Admin API 配置，所有节点都能同步感知到变化。既然 Kong 以 Pod 的形式运行在 k8s 集群中，那么其本身需要对外暴露，这样外部流量才能进来，在本地可以 nodePort 或者 hostNetwork 对外提供服务，在云平台一般通过 LoadBalancer 实现。一般的部署最佳实践是将 Kong 的 Admin 管理功能独立出来一个 Pod，专门作为所有其他节点的统一配置管理，不对外提供流量转发服务，只提供配置功能，而其他 Kong 节点专门提供流量转发功能。

说一说 Kong Ingress Controller：其实没有 Kong Ingress Controller 也完全够用，其存在的意义是为了实现 k8s Ingress 资源对象。我们知道 Ingress 只不过是定义了一些流量路由规则，但是光有这个路由规则没用，需要 Ingress Controller 来将这些路由规则转化成相应代理的实际配置，比如 Kong Ingress Controller 就可以将 Ingress 转化成 Kong 的配置。与 Nginx Ingress Controller 不同，Kong Ingress Controller 不对外提供服务，只作为 k8s Ingress 资源的解析转化服务，将解析转化的结果（Kong 的配置：比如 Service、Route 实体等）通过与 Kong Admin API 写入 Kong 数据库，所以 Kong Ingress Controller 需要和 Kong Admin API 打通。所以当我们需要配置 Kong 的路由时，既可以通过创建 k8s Ingress 实现，也可以通过 Kong Admin API 直接配置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124328579.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)


### helm 部署 Kong（包含 Kong Ingress Controller）

说明：本地集群部署，为了方便 Kong Proxy 和 Kong Admin 没有独立开，共用一个进程，同时提供流量转发和 Admin 管理 API。
使用 helm 官方 Chart: stable/kong，由于我是在本地裸机集群部署，很多云的功能不支持，比如：LoadBalancer、PV、PVC 等，所以需要对 Chart 的 values 文件做一些定制化以符合本地需求：

- 由于本地裸机集群不支持 LoadBalancer，所以采用 nodePort 方式对外暴露 Kong proxy 和 Kong admin 服务，Chart 默认是 nodePort 方式，在这里自定义下端口：Kong proxy 指定 nodePort 80 和 443 端口，Kong Admin 指定 8001 端口：`Values.proxy.http.nodePort: 80 ``Values.proxy.tls.nodePort: 443`, `Values.admin.nodePort: 8001`；
> 注意：默认 k8s nodePort 端口范围是 30000~32767，手动分配该范围之外的端口会报错！该限制可以调整，具体见之前文章：Kubernetes 调整 nodePort 端口范围

- 启用 Kong admin 和 Kong proxy Ingress，部署时会创建相应的 Ingress 资源，实现服务对外访问：`Values.admin.ingress.enabled: true`, `Values.proxy.ingress.enabled: true`，另外还得设置对外访问的域名（没有域名的话可以随便起个域名，然后绑 /etc/hosts 访问）：`Values.admin.ingress.hosts: [admin.kong.com]`, `Values.proxy.ingress.hosts: [proxy.kong.com]`；

- 作为练习，为了方便，Kong admin 改用监听 HTTP 8001 端口：`Values.admin.useTLS: false`, `.Values.admin.servicePort: 8001`, `.Values.admin.containerPort: 8001`。另外也需要将 Pod 探针协议也改为 HTTP：`Values.livenessProbe.httpGet.scheme: HTTP`, `Values.readinessProbe.httpGet.scheme: HTTP`；

- Kong proxy Ingress 启用 HTTPS，这样后续 kong 就可以同时支持 HTTP 和 HTTP 代理了，这里展开下具体过程：

1. 创建 TLS 证书：域名为 proxy.kong.com
```sh
openssl req -x509 -nodes -days 65536 -newkey rsa:2048 -keyout proxy-kong.key -out proxy-kong.crt -subj "/CN=proxy.kong.com/O=proxy.kong.com"
```
2. 使用生成的证书创建 k8s Secret 资源：
```sh
kubectl create secret tls proxy-kong-ssl --key proxy-kong.key --cert proxy-kong.crt -n kong
```
编辑 values 文件启用 Kong Proxy Ingress tls，引用上面创建的 Secret：`Values.proxy.ingress.tls`:
```yaml
- hosts:
      - proxy.kong.com
      secretName: proxy-kong-ssl
```
- 启用 Kong Ingress Controller，默认是不会部署 Kong Ingress Controller：ingressController.enabled: true；

- 由于本地裸机环境不支持 PV 存储，所以在部署时禁用 Postgres 数据持久化：helm 安装时指定 `--set postgresql.persistence.enabled=false`，这样 Postgres 存储会使用 emptyDir 方式挂载卷，在 Pod 重启后数据会丢失，本地自己玩的话可以先这么搞。当然要复杂点的话，可以自己再搭个 nfs 支持 PV 资源对象。

定制后的 values 文件在这里：https://raw.githubusercontent.com/qhh0205/helm-charts/master/kong-values.yml

**helm 部署**
```sh
helm install stable/kong --name kong --set postgresql.persistence.enabled=false -f https://raw.githubusercontent.com/qhh0205/helm-charts/master/kong-values.yml --namespace kong
```
验证部署
```sh
[root@master kong]# kubectl get pod -n kong
NAME                                   READY   STATUS      RESTARTS   AGE
kong-kong-controller-76d657b78-r6cj7   2/2     Running     1          58s
kong-kong-d889cf995-dw7kj              1/1     Running     0          58s
kong-kong-init-migrations-c6fml        0/1     Completed   0          58s
kong-postgresql-0                      1/1     Running     0          58s

[root@master kong]# kubectl get ingress -n kong
NAME              HOSTS            ADDRESS   PORTS     AGE
kong-kong-admin   admin.kong.com             80        84s
kong-kong-proxy   prox.kong.com              80, 443   84s
```
curl 测试
```sh
[root@master kong]# curl -I http://admin.kong.com
HTTP/1.1 200 OK
Content-Type: application/json
...

[root@master kong]# curl http://proxy.kong.com
{"message":"no Route matched with those values"}
```
### 部署 Konga

上面已经将整个 Kong 平台运行在了 Kubernetes 集群，并启用了 Kong Ingress Controller，但是目前做 Kong 相关的路由配置只能通过 curl 调 Kong Admin API，配置起来不是很方便。所以需要将针对 Kong 的 UI 管理服务 Konga 部署到集群，并和 Kong 打通，这样就可以可视化做 Kong 的配置了。由于 Konga 的部署很简单，官方也没有 Chart，所以我们通过 yaml 文件创建相关资源。

为了节省资源，Konga 和 Kong 共用一个 Postgresql，Konga 和 Kong 本身对数据库资源占用很少，所以两个同类服务共用一个数据库是完全合理的。下面为 k8s 资源文件，服务对外暴露方式为 Kong Ingress，域名设为(名字随便起的，绑 host 访问)：konga.kong.com：
```sh
//数据库密码在前面安装 Kong 时 Chart 创建的 Secret 中，获取方法：
kubectl get secret kong-postgresql -n kong -o yaml | grep password | awk -F ':' '{print $2}' | tr -d ' ' | base64 -d
```
konga.yml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: konga
  name: konga
spec:
  replicas: 1
  selector:
    matchLabels:
      app: konga
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: konga
    spec:
      containers:
      - env:
        - name: DB_ADAPTER
          value: postgres
        - name: DB_URI
          value: "postgresql://kong:K9IV9pHTdS@kong-postgresql:5432/konga_database"
        image: pantsel/konga
        imagePullPolicy: Always
        name: konga
        ports:
        - containerPort: 1337
          protocol: TCP
      restartPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: konga
spec:
  ports:
  - name: http
    port: 1337
    targetPort: 1337
    protocol: TCP
    selector:
    app: konga

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: konga-ingress
spec:
  rules:
  - host: konga.kong.com
    http:
      paths:
      - path: /
        backend:
          serviceName: konga
          servicePort: 1337
```

**kubectl 部署 Konga**
```sh
kubectl create -f konga.yml -n kong
```
部署完成后绑定 host 将 konga.kong.com 指向集群节点 IP 即可访问：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124512782.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

接下来随便注册个账号，然后配置连接到 Kong Admin 地址，由于都在集群内部，所以直接用 Kong Admin 的 ServiceName + 端口号连就可以：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124534491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

连接没问题后，主页面会显示 Kong 相关的全局信息：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124559767.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)


示例：通过 Konga 配置对外访问 Kubernetes Dashboard

之前我们基于 Nginx Ingress Controller 对外暴露 Kubernetes Dashboard，现在我们基于集群中 Kong 平台配置对外访问，通过 Konga 可视化操作。
通过 Konga 配置服务对外访问只需要两步：

创建一个对应服务的 Service（不是 k8s 的 Servide，是 Kong 里面 Service 的概念：反向代理上游服务的抽象）；
创建对应 Service 的路由；
下面以配置 Kubernetes dashboard 服务对外访问为例，对外域名设为 dashboard.kube.com (名字随便起的，绑 host 访问)

1. 创建 Kong Service：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124622663.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

1. 创建服务路由：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124640678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124659975.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124719509.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)

配置完成，浏览器测试访问：https://dashboard.kube.com
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190817124734862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FpYW5naGFvaGFv,size_16,color_FFFFFF,t_70)
### 相关文档

https://konghq.com/solutions/kubernetes-ingress/ | Kong on Kubernetes
https://konghq.com/blog/kubernetes-ingress-controller-for-kong/ | Announcing the Kubernetes Ingress Controller for Kong
https://docs.konghq.com/install/kubernetes/ | Kong and Kong Enterprise on Kubernetes
https://github.com/Kong/kubernetes-ingress-controller | GitHub Kong Ingress Controller
