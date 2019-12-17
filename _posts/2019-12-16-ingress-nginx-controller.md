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
