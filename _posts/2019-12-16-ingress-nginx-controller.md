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
自从公司所有服务迁移到k8s之后，快站的c端页面打开就很少能够使用到缓存