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
![检查](https://xhb3909.github.io/img/custom/jiancha.png)
红色标注的地方`x-cache-status: MISS`证明没有击中缓存，还是直接调用的后端服务来获取页面信息。如果`x-cache-status: HIT`则代表击中缓存。
这时页面的响应速度会有比较大的提升。为了让读者能够更全面的了解整个调用链路，这里我来展示一下我们公司的服务部署结构图
![快站服务](https://xhb3909.github.io/img/custom/kuaizhanjiagou.png)
这个图里面，我简单说一下C端页面的一个访问流程，访问c端链接，
经过外网负载均衡`kzno`(这个项目是`nginx+openresty`主要承担快站的外网和内网的路由分发、频率限制等功能)
访问到gateway(这是用go写的一个网关服务，主要的作用是鉴权、域名识别、转发服务等功能)。gateway通过`k8s service dns name` 比如
`html-render.kuaizhan-cl.svc.cluster.local:80`把当前的请求转发到c端的服务html-render，服务里面部署的是`nginx + PHP`的组合。
html-render服务使用的是`nginx`内置的缓存，具体配置如下:
![cache](https://xhb3909.github.io/img/custom/cache.png)
如果100s之内访问过某个容器，就会在当前容器里面保存有当前请求域名+路径的缓存值，所以下次如果还能请求到这个容器，就可以利用上`nginx`缓存。
从而，不必在upstream到上游服务。
因为快站c端的qps比较高，所以针对html-render部署的pod容器个数就会很多，这样同一个请求其实会被打散到不同的容器里，所以根本不能很好的利用上
缓存。😿

### 一致性hash
我先来讲讲一致性hash是怎么一回事吧，毕竟这篇文章和一致性hash有很大的关系。一致性Hash算法使用取模的方法，取模法针对`nginx upstream`的数量
进行取模，而一致性Hash算法是对2^32取模，什么意思呢？简单来说，一致性Hash算法将整个哈希值空间组织成一个虚拟的圆环，
如假设某哈希函数H的值空间为0-2^32-1（即哈希值是一个32位无符号整形），整个哈希环如下：
![hash-0](https://xhb3909.github.io/img/custom/hash-0.jpg)
整个空间按顺时针方向组织，圆环的正上方的点代表0，0点右侧的第一个点代表1，
以此类推，2、3、4、5、6……直到2^32-1，
也就是说0点左侧的第一个点代表2^32-1， 0和2^32-1在零点中方向重合，我们把这个由2^32个点组成的圆环称为Hash环。

服务器这时可以使用Hash进行一个哈希，
具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置，如下图所示
![hash-1](https://xhb3909.github.io/img/custom/hash-1.jpg)
使用如下算法定位数据访问到相应服务器：
将数据key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，第一台遇到的服务器就是其应该定位到的服务器！
如下图所示
![hash-2](https://xhb3909.github.io/img/custom/hash-2.jpg)

如果某台机器宕机。受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响。
相应地新增一台机器。受影响的数据仅仅是新服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它数据也不会受到影响。

这么一看一致性hash的好处就可见一斑。但是我们其实还是需要关注热点问题。服务节点太少时，容易因为节点分部不均匀而造成数据倾斜。严重的话，
可能会存在连锁反应造成雪崩。

为了解决这种数据倾斜问题，一致性Hash算法引入了虚拟节点机制，
即对每一个服务节点计算多个哈希，每个计算结果位置都放置一个此服务节点，称为虚拟节点。具体做法可以在服务器IP或主机名的后面增加编号来实现。
如下图所示
![hash-3](https://xhb3909.github.io/img/custom/hash-3.jpg)

这样我们就可以比较完美的解决热点问题。对于`nginx`来说需要在upstream里指定weight字段，作为server权重，对应虚拟节点数目。
![upstream-ex](https://xhb3909.github.io/img/custom/upstream-ex.png)

### 浅谈控制器模式
在讲解今天的主角`ingress-nginx-controller`之前，我很想和大家分享一下k8s的控制器。我们要懂得控制器的一个大概运作原理，才能更好
地理解`ingress-nginx-controller`. (这段内容参考极客时间张磊大神分享的自定义控制器，感兴趣的可以去看看). 话不多说，先来个图:
[!k8s-controller](https://xhb3909.github.io/img/custom/k8s-controller.png)
控制器要做的第一件事，是从 Kubernetes 的 APIServer 里获取它所关心的对象，比如Ingress。
APIServer 是master节点下，负责API服务的。以及集群的持久化数据,也都是由APIServer处理之后保存到`etcd`中。
[!master](https://xhb3909.github.io/img/custom/master.png)
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

### `ingress-nginx-controller`
千呼万唤始出来，我们的主角终于闪亮登场了。



