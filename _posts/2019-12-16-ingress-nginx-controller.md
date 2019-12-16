---
layout:     post
title:      ingress-nginx-controller
date:       2019-12-16
author:     xhb
header-img: img/post-bg-debug.png
catalog: true
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