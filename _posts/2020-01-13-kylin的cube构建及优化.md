---
layout:     post
title:      kylin的cube构建及优化
date:       2020-01-13
author:     xhb
header-img: img/post-bg-ioses.jpg
catalog: true
sid: 20200113
tags:
    - kylin
    - hadoop
---

# 开篇
在[腾讯云emr部署kylin](https://xhb3909.com/2020/01/12/%E8%85%BE%E8%AE%AF%E4%BA%91%E5%BC%B9%E6%80%A7mapreduce%E9%83%A8%E7%BD%B2kylin/)
这一篇讲解了在腾讯云emr上如何部署`kylin`，这一篇就基于部署好的`kylin`构建cube。

# 具体实操

首先在`kylin`管理界面上新建项目，我新建了一个meter_v2的项目，并且用户可以针对这个项目去点击project config
来配置`kylin`的参数，`kylin`的参数具体有哪些可以参考[kylin官方文档](http://kylin.apache.org/cn/docs/install/configuration.html)
* ![新建项目](https://pic.kuaizhan.com/g3/9b/c1/0ef8-e8ae-4989-9554-793f1f612d3a45)

新建成功之后，界面如下所示:
* ![page_empty](https://pic.kuaizhan.com/g3/0b/78/3ec9-c694-4ee9-a2fb-02024966237226)

接下来我们需要把hive中的表数据导入到`kylin`中来，需要点击`Load Table From Tree`按钮，然后同步你当前想要导入的hive表

* ![sync table](https://pic.kuaizhan.com/g3/b1/fc/6165-6729-431e-92b8-7fd9031e1b9d85)

在这里我把meteor库中的`pageview`表同步到hive中，同步的过程中会触发一个MapReduce任务，用来计算各个列的基数
为多少，计算成功之后在`kylin`中展示如下
* ![compute_success](https://pic.kuaizhan.com/g3/a3/4a/c3dd-d0f9-4056-a06c-5f4bc31b440c42)

页面链接DL的基数为48823，所以也就是说明如果按照DL维度来进行聚合的话，最多只需要存储48823条数据
这里我把新建`pageview`表的语句展示一下:

```java

    public void createTable(String tableName) {
        sparkSession.sql("use " + databaseName);
        sparkSession.sql("CREATE TABLE IF NOT EXISTS " + tableName +
                " (" +
                "tid STRING," +
                "dl STRING," +
                "dt STRING," +
                "r STRING," +
                "rt STRING," +
                "ua STRING," +
                "ct STRING," +
                "br STRING," +
                "tm BIGINT," +
                "nu INT," +
                "uid STRING," +
                "hour INT," +
                "d1 STRING," +
                "d2 STRING," +
                "d3 STRING," +
                "d4 STRING," +
                "d5 STRING," +
                "d6 STRING," +
                "d7 STRING," +
                "d8 STRING," +
                "d9 STRING," +
                "d10 STRING" +
                ") " +
                "PARTITIONED BY" +
                " (" +
                ds + " STRING) " +
                "STORED AS ORC");
    }

```

每个字段的含义分别如下

```
`pageview表`

* tid: 站点id 
* dl: 页面链接 
* dt: 页面title
* r: referer
* rt: referer type
* ua: user-agent
* ct: mobile/pc/pad
* br: uc、qq、wx、chrome
* tm: 事件发生时间戳
* nu: 新用户 
* uid: uid
* ds: 日期
* hour: 小时
* d1: 自定义参数1 
* d2: 自定义参数2 
* d3: 自定义参数3 
* d4: 自定义参数4 
* d5: 自定义参数5 
* d6: 自定义参数6 
* d7: 自定义参数7 
* d8: 自定义参数8
* d9: 自定义参数9
* d10: 自定义参数10

```

## model

我们就基于这个表来创建数据模型。数据模型(Data Model)是Cube的基础，根据分析需求进行设计。有了数据模型之后，定义cube的时候就可以直接从此模型
定义的表和列中进行选择了。基于一个数据模型可以创建多个cube，减少用户的重复性工作。

首先，输入model的名称:

* ![model_1](https://pic.kuaizhan.com/g3/65/bb/55d3-6f8c-4439-996a-ba0baeb0671326)

然后，选择事实表，这里我选择的事实表是meteor库下的`pageview`表，如下所示:

* ![model_2](https://pic.kuaizhan.com/g3/85/b0/0896-4945-40d5-8ce5-9d1c97ef708383)

下一步是选择这个表中的哪些列作为维度，这里只是一个选择范围，不代表这些列将来一定要用作Cube的维度和度量，你可以把所有可能会用到的列都选进来，
后续创建cube的时候，将只能从这些列中进行选择，这里我选择了站点id(TID)、页面链接(DL)、页面title(DT)、referer(r)、referer type(rt)、
user-agent(ua)、终端类型(ct)、浏览器内核(br)、日期(ds)、小时(hour)作为维度。

* ![model_3](https://pic.kuaizhan.com/g3/89/b5/ffd2-07c0-428a-86ab-639d4308c44d14)

接下来是选择这个表中的哪些列作为度量列，度量只能来自事实表或不加载进内存的维度表，这里我选择新用户(NU)、用户id(uid)来作为度量的指标:

* ![model_4](https://pic.kuaizhan.com/g3/a2/de/10e8-02d4-429d-9f7e-afd79923208c76)

最后一步，是为模型补充分隔时间列信息和过滤条件。如果此模型中的事实表记录是按照时间增长的，那么可以指定一个日期/时间列作为模型的分隔时间列，
从而可以让cube按此列做增量构建。因为我们这个表是用来做数据统计的，数据按照日期(DS)增长，所以我们可以选择按照DS为分隔列，来增量构建cube.
具体配置如下:

* ![model_5](https://pic.kuaizhan.com/g3/c0/05/06fe-7a2a-4b43-9b33-ef3694ef6f7175)

经过上述几步的配置，我们就完成了一个model的创建。

## cube
接下来就基于上面的model创建cube。

首先，输入cube的名称:
* ![cube_1](https://pic.kuaizhan.com/g3/a5/89/4e05-274e-4694-b0d5-1d024a63b82270)

接下来，添加cube的维度，这里我们使用站点id(TID)、页面链接(DL)、refer(R)

