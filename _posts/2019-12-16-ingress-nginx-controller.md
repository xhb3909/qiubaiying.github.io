---
layout:     post
title:      ingress-nginx-controller
date:       2019-12-16
author:     xhb
header-img: img/post-bg-debug.png
catalog: true
sid: 20191216
tags:
    - kubernetes
---

# `nginx`一致性hash在k8s中的实践

今天，我想跟大家分享的是，工作中，使用`ingress-nginx-controller`来实践一致性hash的辛酸历程.避免以后的人少走弯路.😭

### 背景
自从公司所有服务迁移到k8s之后，快站的c端页面打开就很少能够使用到页面缓存。打开`https://yongyong.kuaizhan.com`，谷歌浏览器里打开检查，
就会发现，调用如下:
![检查](https://pic.kuaizhan.com/g3/24/67/99f8-2f7f-44b9-89cb-89dec75c4f6323)
红色标注的地方`x-cache-status: MISS`证明没有击中缓存，还是直接调用的后端服务来获取页面信息。如果`x-cache-status: HIT`则代表击中缓存。
这时页面的响应速度会有比较大的提升。为了让读者能够更全面的了解整个调用链路，这里我来展示一下我们公司的服务部署结构图
![快站服务](https://pic.kuaizhan.com/g3/48/cf/c75c-efda-4863-a5de-19d2f95d8ef236)
这个图里面，我简单说一下C端页面的一个访问流程，访问c端链接，
经过外网负载均衡`kzxx`(这个项目是`nginx+openresty`主要承担快站的外网和内网的路由分发、频率限制等功能)
访问到gateway(这是用go写的一个网关服务，主要的作用是鉴权、域名识别、转发服务等功能)。gateway通过`k8s service dns name` 比如
`html-xx.kuaizhan-xx.svc.cluster.local:80`把当前的请求转发到c端的服务html-xx，服务里面部署的是`nginx + PHP`的组合。
html-render服务使用的是`nginx`内置的缓存，具体配置如下:
![cache](https://pic.kuaizhan.com/g3/97/c4/fcbe-f021-4ce0-ac01-ac483c397a2038)
如果100s之内访问过某个容器，就会在当前容器里面保存有当前请求域名+路径的缓存值，所以下次如果还能请求到这个容器，就可以利用上`nginx`缓存。
因为快站c端的qps比较高，所以针对html-render部署的pod容器个数就会很多，这样同一个请求其实会被打散到不同的容器里，所以根本不能很好的利用上
缓存。😿

### 一致性hash
我先来讲讲一致性hash是怎么一回事吧，毕竟这篇文章和一致性hash有很大的关系。一致性Hash算法使用取模的方法，取模法针对`nginx upstream`的数量
进行取模，而一致性Hash算法是对2^32取模，什么意思呢？简单来说，一致性Hash算法将整个哈希值空间组织成一个虚拟的圆环，
如假设某哈希函数H的值空间为0-2^32-1（即哈希值是一个32位无符号整形），整个哈希环如下：
![hash-0](https://pic.kuaizhan.com/g3/e4/5d/be20-92c4-42aa-a88a-b9da6145f1a009)
整个空间按顺时针方向组织，圆环的正上方的点代表0，0点右侧的第一个点代表1，
以此类推，2、3、4、5、6……直到2^32-1，
也就是说0点左侧的第一个点代表2^32-1， 0和2^32-1在零点中方向重合，我们把这个由2^32个点组成的圆环称为Hash环。

服务器这时可以使用Hash进行一个哈希，
具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置，如下图所示
![hash-1](https://pic.kuaizhan.com/g3/1a/f7/1c6e-5ed4-4095-bc68-1fca001cfd9945)
使用如下算法定位数据访问到相应服务器：
将数据key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，第一台遇到的服务器就是其应该定位到的服务器！
如下图所示
![hash-2](https://pic.kuaizhan.com/g3/b1/64/6de3-7403-4077-ad7d-148b56618eae07)

如果某台机器宕机。受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响。
相应地新增一台机器。受影响的数据仅仅是新服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它数据也不会受到影响。

这么一看一致性hash的好处就可见一斑。但是我们其实还是需要关注热点问题。服务节点太少时，容易因为节点分部不均匀而造成数据倾斜。严重的话，
可能会存在连锁反应造成雪崩。

为了解决这种数据倾斜问题，一致性Hash算法引入了虚拟节点机制，
即对每一个服务节点计算多个哈希，每个计算结果位置都放置一个此服务节点，称为虚拟节点。具体做法可以在服务器IP或主机名的后面增加编号来实现。
如下图所示
![hash-3](https://pic.kuaizhan.com/g3/8e/7e/c278-8d6c-4225-9c7e-c43e1361767873)

这样我们就可以比较完美的解决热点问题。对于`nginx`来说需要在upstream里指定weight字段，作为server权重，对应虚拟节点数目。
![upstream-ex](https://pic.kuaizhan.com/g3/11/8e/9340-3e1f-43e4-bc77-636dabcaa0e495)

### 浅谈控制器模式
在讲解今天的主角`ingress-nginx-controller`之前，我很想和大家分享一下k8s的控制器。我们要懂得控制器的一个大概运作原理，才能更好
地理解`ingress-nginx-controller`. (这段内容参考极客时间张磊大神分享的自定义控制器，感兴趣的可以去看看). 话不多说，先来个图:
![k8s-controller](https://pic.kuaizhan.com/g3/e6/8e/e4b9-c4a2-41c8-a749-508c217d6a3897)
控制器要做的第一件事，是从 Kubernetes 的 APIServer 里获取它所关心的对象，比如Ingress。
APIServer 是master节点下，负责API服务的。以及集群的持久化数据,也都是由APIServer处理之后保存到`etcd`中。
![master](https://pic.kuaizhan.com/g3/9d/38/9a17-b2ec-40bb-8fe7-d1555d71eba810)
其余两个组件 `kube-scheduler`: 负责容器调度, `kube-controller-manager`: 负责容器编排。
继续说控制器. 获取对象的操作，依靠的是一个叫作 Informer（通知器）的代码库完成的。
* Informer的第一个职责是同步本地缓存。
Informer 与 API 对象是一一对应的，所以传递给自定义控制器的，正是一个 Ingress 对象的 Informer（Ingress Informer).
Ingress Informer 使用 ingressClient，跟 APIServer 建立了连接。不过，真正负责维护这个连接的，则是 Informer 所使用的 Reflector 包。
Reflector 使用的是一种叫作 ListAndWatch 的方法，来“获取”并“监听”这些 Ingress 对象实例的变化。
在 ListAndWatch 机制下，一旦 APIServer 端有新的 Ingress 实例被创建、删除或者更新，
Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为增量（Delta），
它会被放进一个 Delta FIFO Queue（增量先进先出队列）中。而另一方面，Informer 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。
每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 Store。

* Informer的第二个职责，则是根据这些事件的类型，触发事先注册好的 ResourceEventHandler。
这些 Handler，需要在创建控制器的时候注册给它对应的 Informer。所以操作API对象的代码逻辑是在handler里实现的。
而具体的处理操作，都是将该事件对应的 API 对象加入到工作队列中。

所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client。
它是自定义控制器跟 APIServer 进行数据同步的重要组件。更具体地说，Informer 通过一种叫作 ListAndWatch 的方法，
把 APIServer 中的 API 对象缓存在了本地，并负责更新和维护这个缓存。

接下来控制循环(Control Loop)，则会不断地从这个工作队列里拿到这些 Key，然后开始执行真正的控制逻辑。
控制循环会等待 Informer 完成一次本地缓存的数据同步操作；并且直接通过 `goroutine` 启动一个（或者并发启动多个）“无限循环”的任务。
真正处理API对象的逻辑则在循环当中完成。

所以控制器其实是根据informer的reflector来监听并更新API对象，并通过EventHandler处理好对象放入worker queue中，最终由
(control loop)从工作队列中取到key，执行真正的逻辑操作。

* 这时候我想问大家一个问题，为什么 Informer 和自定义控制器的控制循环之间，一定要使用一个工作队列来进行 ?
我觉得主要的原因在于 Informer 和 控制循环分开是为了解耦，防止控制循环执行过慢把Informer拖死。
这样的设计也是为了匹配双方速度不一致，有一个中间的队列来做协调.

### `ingress-nginx-controller` 介绍
千呼万唤始出来，我们的主角终于闪亮登场了。
官方网址`https://kubernetes.github.io/ingress-nginx/`
首先来说一下`nginx`控制器是如何工作的

#### `nginx configuration`
Ingress控制器的目标是汇编配置文件（`nginx.conf`）。主要目的是在配置文件中进行任何更改后需要重新加载`NGINX`. 
需要特别注意的是，我们不会在仅影响上游配置的更改中重新加载`Nginx`（即，在部署应用程序时端点EndPoints更改）。
因为使用了`lua-nginx-module`实现这一目标

#### `Building the NGINX model`
* 按CreationTimestamp字段对Ingress规则进行排序，即先按旧规则。
* 如果在多个Ingress中为同一主机定义了相同路径，则最早的规则将获胜。
* 如果多个Ingress包含同一主机的TLS部分，则最早的规则将获胜。
* 如果多个Ingress定义了一个影响Server块配置的注释，则最早的规则将获胜。
* 创建`NGINX`服务器列表（按主机名）
* 创建`NGINX`上游列表
* 如果多个入口为同一主机定义了不同的路径，则入口控制器将合并这些定义。
* 注释将应用于Ingress中的所有路径。
* 多个Ingress可以定义不同的注释。这些定义在Ingress之间不共享。

#### `When a reload is required`
* 创建新的Ingress resource
* TLS部分已添加到现有Ingress
* Ingress annotation的更改不仅影响上游配置，而且影响更大。例如，负载平衡注释不需要重新加载。
* 从Ingress添加/删除路径。
* Ingress, Service, Secret 被删除
* 可以从Ingress获得一些缺少的引用对象，例如Service或Secret。
* Secret更新

#### `Avoiding reloads`
在某些情况下，可以避免重新加载，尤其是在端点发生更改时.例如: pod 重启或被替换.

##### `Avoiding reloads on Endpoints changes`
在每个端点更改上，控制器从其看到的所有服务中获取端点并生成相应的Backend对象。然后，将这些对象发送到在`Nginx`内部运行的Lua处理程序。 
Lua代码又将这些后端存储在共享内存区域中。然后，对于在`balancer_by_lua`上下文中运行的每个请求，
Lua代码将检测到应该从哪个端点选择上游对等方，并应用配置的负载均衡算法来选择对等方。
在具有频繁部署应用程序的相对较大的集群中，此功能节省了大量`Nginx`重载，否则可能会影响响应延迟.

说了这么多. 总结一下，`nginx`控制器随着ingress的更改，而需要重新reload。但是为了避免这种reload操作，我们可以使用ConfigMap更改配置。或者
当endpoints修改的时候，控制器会根据`lua-nginx-module`来自动更新endpoints。

### `ingress-nginx-controller` 部署
首先来看一下`deployment.yaml`文件:
```YAML
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.25.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
---

```

看了这么多配置，是不是心里一慌。别怕，我来简单解释一下这个yaml文件要做什么:
* ConfigMap: 允许您将配置工件与image内容分离，以使容器化的应用程序具有可移植性。简单说，就是可配置，即使pod处于运行状态，也可以
修改ConfigMap来避免`reload nginx`
* `RBAC`: 基于角色的访问控制（Role-Based Access Control)
* ClusterRole: 分配给角色的权限适用于整个集群
* ClusterRoleBinding: 将ClusterRole绑定到特定帐户
* Role: 分配给适用于特定名称空间的角色的权限
* RoleBinding: 将角色绑定到特定帐户
* 为了将`RBAC`应用于`nginx-ingress-controller`，应将该控制器分配给ServiceAccount.
该ServiceAccount应该绑定到为`nginx-ingress-controller`定义的Roles和ClusterRoles

分配这个权限的目的是为了使`nginx-ingress-controller`能够充当跨集群的入口。
这些权限被授予名为`nginx-ingress-clusterrole`的ClusterRole
* `configmaps`, endpoints, nodes, pods, secrets: list, watch
*  nodes: get
*  services, ingresses: get, list, watch
*  events: create, patch
*  ingresses/status: update
* Deployment: 管理`ingress nginx`pod的控制器，根据这个我们能知道绝大多数控制器本质部署的是pod

上面的文件配置完，需要配置service yaml

```YAML

kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: tcp-80-80-cluster
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  sessionAffinity: None

```
> 通过clusterIp的方式部署service

* 公司的k8s集群是在腾讯云搭建的版本是v1.13，所以执行以下命令来创建`ingress-nginx-controller`

```shell
kubectl create namespace ingress-nginx
kubectl apply -f ingress-nginx-deployment.yaml
kubectl apply -f ingress-nginx-service.yaml 
```
执行上述三个命令，就在k8s集群里新建了一个ingress 控制器
因为默认一个ingress，只能指定一个控制器来控制它的一个声明周期，所以我们看看ingress的配置

```YAML
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuaizhan-wap
  namespace: kuaizhan-cl
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$host$request_uri"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"

spec:
  rules:
  - host:
    http:
      paths:
      - path: /
        backend:
          serviceName: html-render-pre
          servicePort: 80
```
> 这个yaml文件配置了关于控制器的三个参数
* `kubernetes.io/ingress.class` 指定的是`nginx`控制器，默认在腾讯云部署的值是`qcloud`是腾讯云定制的一个控制器
* `nginx.ingress.kubernetes.io/upstream-hash-by` 指定一致性hash的一个规则，是按照当前域名+路径进行hash
* `nginx.ingress.kubernetes.io/ssl-redirect` 禁用308的重定向请求

> 执行ingress
```shell
kubectl apply -f html-xx-ingress.yaml 
```

> 执行这个命令的同时，`ingress nginx`控制器会自动更新当前ingress的配置，
```shell
kubectl exec -n ingress-nginx nginx-ingress-controller-5cd5655c57-fjq7f cat /etc/nginx/nginx.conf
```
> 使用如下命令看到的配置文件
![linux-1](https://pic.kuaizhan.com/g3/6a/8a/79ab-9ba3-402c-be0e-e46519b367c618)
> 图中我们看出配置的规则已经生效了，当访问路径的时候直接请求到html-render-pre的后端服务，为了测试一致性hash，html-render-pre的容器个数
设置为5个

接下来要模拟多个请求访问服务，看看配置的一致性hash是否生效，这里使用http_load工具，因为它支持多个不同url的请求来并发访问服务
```shell
http_load -rate 100 -seconds 10 url.log
-rate 简写-p ：含义是每秒的访问频率
-seconds简写-s ：含义是总计的访问时间
url.log里面写了很多个快站c端的url
```
> 执行过程中发现了一个很奇怪的现象，就是所有的请求，并没有均匀分配，而是都打到了同一个节点。
奇怪，明明配置了一致性hash的参数是`$host$request_uri`，但是感觉没有生效。

### 源码探究`ingress-nginx-controller`的一致性hash奥秘
不得不看看源码如何处理的一致性hash。首先我观测到`nginx.conf`文件中关于upstream的代码
![linux-2](https://pic.kuaizhan.com/g3/20/d5/ce10-e01c-4217-b75f-321102cf09ee50)
发现它是通过lua动态获取的endpoints数据，调用了balance.balance()的代码，于是我在github上下载了`ingress-nginx`的项目，找到了
balance.lua文件，这里执行的获取balance的操作
![lua-0](https://pic.kuaizhan.com/g3/53/9c/8fae-6f14-4a56-a868-0891d95c279699)
定位到balance其实通过就是implement对象，具体代码如下所示
![lua-1](https://pic.kuaizhan.com/g3/32/bd/34f7-398e-4745-9d1b-13b2cb80a43b48)
发现了很重要的一点，如果选择的是一致性hash，则会使用`chash`这个对象。接下来，则全局搜索`chash`，发现正是一致性hash代码实现的核心逻辑
![lua-2](https://pic.kuaizhan.com/g3/76/0b/8c1d-55d1-4459-bd95-f774ae10761745)
重点就是balance方法，调用了util模块下的lua_ngx_var，我们来看一下这块代码
![lua-3](https://pic.kuaizhan.com/g3/51/78/d69e-f613-4d7a-8f16-34566dcc082003)
发现问题了没有，原来一致性hash只支持传入单个变量，如果我传入的是$host$request_uri，最后会给我变成host$request_uri，这个变量，通过ngx.var
获取不到任何值!!

找到了问题关键，本来想在github上针对`ingress-nginx`项目提一个issue，但是一搜索发现已经有人提出了这个问题，并且有一个merge request正是
关于这块逻辑的优化，支持多个参数的一致性hash，不过最终没有通过代码提审。感兴趣的伙伴可以看看这个issue
* https://github.com/kubernetes/ingress-nginx/issues/3458
* https://github.com/kubernetes/ingress-nginx/pull/3460/commits/69451b69323062d4847db3864863af7d1b77f9cb

最终把$host$request_uri改成$host就可以均匀分配到每个节点。

### 部署方式还有问题

但其实这样的配置还是有问题的，就是当服务的节点数增多，其实就会和我上文讲到的一致性hash原理那样，同样存在热点问题。
这时候想到的方式就是控制器有没有类似weight的参数配置，但是翻阅整个文档，都没有发现有这个参数的配置，只好查看源码。发现
![lua-4](https://pic.kuaizhan.com/g3/0a/8f/57f5-7b3b-4c89-a105-3d9083cde1af45)代码里默认配置了weight为1。这样只能后端服务部署多个节点。
来解决没有虚拟节点带来的热点问题。

但是别急，`nginx`控制器还是想到了这一点
官方文档描述
```
"subset" hashing can be enabled setting nginx.ingress.kubernetes.io/upstream-hash-by-subset: "true". 
This maps requests to subset of nodes instead of a single one. 
upstream-hash-by-subset-size determines the size of each subset (default 3).

```
大致意思是说，使用子集的方式。这会将请求映射到节点的子集，而不是单个请求。子集大小的上游哈希值确定每个子集的大小（默认为3）。这样其实就可以
在一致性hash和热点之间做出一个比较好的平衡.

如果只是单纯的想用负载均衡的功能，可以指定`nginx.ingress.kubernetes.io/service-upstream`为true。
因为默认情况下，`NGINX`入口控制器使用`NGINX`上游配置中所有端点（Pod IP /端口）的列表。
对于零停机时间部署之类的事情，这可能是理想的，因为它减少了Pod上下时重新加载`NGINX`配置的需求。避免了pod容器个数发生更新，而更新lua的动态内存。

### 尾声
讲了这么多，想必大家对`ingress-nginx-controller`有了一个大概的认识，但是我还是大家如果要使用`nginx`控制器的话，需要多看看文档，
选择适合你们自己公司的一个方案。我们要勇于拥抱新的技术🤗。