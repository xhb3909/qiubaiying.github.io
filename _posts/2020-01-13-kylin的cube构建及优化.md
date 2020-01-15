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

首先，输入cube的名称 (Cube Info):
* ![cube_1](https://pic.kuaizhan.com/g3/a5/89/4e05-274e-4694-b0d5-1d024a63b82270)

接下来，添加cube的维度，这里我们使用站点id(TID)、页面链接(DL)、refer(R)、user-agent(UA)、小时(hour)、日期(DS)、域名(D1)作为维度 (Dimensions)
* ![cube_2](https://pic.kuaizhan.com/g3/9e/11/d30b-22aa-4a73-b94b-248f1feec91b30)

然后，我们可以选择添加某些度量。`kylin`默认会为我们创建一个constant的度量，可以通过count(*)函数统计相关指标。下图展示度量的配置，这里
我要统计distinct uid和sum nu (Measures):
* ![cube_3](https://pic.kuaizhan.com/g3/e4/1c/b9ab-a1e9-4903-982b-81b1da55480e70)

下一步是(Refresh Setting)，这里我们有一些比较特殊的配置。如果当前cube是基于可分区的model设置的，我们就需要关系segment的数量，
因为每次新的cube构建都会生成一个segment，如果segment的数量太多，就会产生大量的碎片, 导致运行时的查询引擎需要聚合多个Segment的结果才能返回
正确的查询结果。从存储引擎的角度来说，大量的segment会带来大量的文件，这些文件会充斥系统为`kylin`所提供的命名空间，给存储空间的多个模块带来
巨大的压力，如Zookeeper、`HDFS Namenode`等。所以基于以上原因，通过以下配置项来帮助管理Segment碎片.
1. Auto Merge Thresholds: 允许用户设置几个层级的时间阈值，层级越靠后，时间阈值越大。举个例子，用户可以为一个cube指定层级(7天、28天)。
每当cube中有新的segment状态变为READY的时候，会触发一次系统自动合并的尝试。系统会尝试最大一级的时间阈值，系统会查看是否能把连续的或干个
Segment合并为一个超过28天的较大segment，在挑选连续segment过程中，如果有的segment本身的时间长度已经超过28天，那么系统会跳过该
segment，当没有满足条件的连续segment能够累计超过28天，系统会使用下一个层级的时间阈值重复此寻找的过程。每当有满足条件的连续Segment被找到，
系统就会触发一次自动合并Segment的构建任务。在构建任务完成后，新的segment被设置为READY状态，自动合并的整个尝试过程则需要重新执行.

2. volatile range, 需要在后面的文本框中输入最近的不想要合并的segment天数。举个例子，如果设置volatile range为2，auto merge thresholds
为7.现有01-01到01-08的数据即8个segment，自动合并不会被开启，直到01-09的数据进来，才会触发自动合并，将01-01到01-07的数据合并为一个
segment。

3. retention threshold. 是清理不再使用的segment。许多业务场景只会对过去一段时间内的数据进行查询。如果只显示过去一年数据的报表，
事实上，只需要保留过去一年的segment. 这种情况下，我们可以将， `retention threshold` 设置为365。每当有新的segment状态变为
READY的时候，系统会检查每一个segment。如果它的结束时间距离最晚的一个segment的结束时间已经大于`retention threshold`。那么这个
segment将被视为无须保留，系统会自动从cube中删除这个segment

配置如下图所示:
* ![cube_4](https://pic.kuaizhan.com/g3/f5/6e/ad8a-1d50-48ac-a4c5-e3878f1edbd679)

高级设置(Advanced Setting)。这个里面的配置还是很重要的。首选先说的是聚合组(Aggregation Groups)。如下图所示:
* ![cube_5](https://pic.kuaizhan.com/g3/63/6f/2885-87f1-4666-8414-d0cf6533c59929)
目前只有一个分组，在某一分组内，首先需要指定这个分组包含哪些维度，然后才可以进行必须维度、层级维度和联合维度的创建。除了Include选项，
其他三项都是可选的。此外，还可以设置`Max Dimension Combination`(默认为0，即不加限制)，该设置表示对聚合组的查询最多包含几个维度。
在生成聚合组时会不生成超过`Max Dimenstion Combination`中设置的数量的Cuboid，因此可以有效减少Cuboid的总数.

如果我们的cube中有一些维度的基数很大，如果不做特殊处理，它会和其他维度进行各种组合，从而产生大量包含它的Cuboid。所有包含高基数维度的
Cuboid在行数和体积上都会非常庞大，这会导致整个cube的膨胀率过大。如果根据业务知道这个高基数的维度只会与若干个维度(而不是所有维度)同时
被查询，就可以通过聚合组对这个高基数维度做一定的隔离。

我们把这个高基数的维度放入一个单独的聚合组，把所有可能会与这个额高基数维度一起被查询到的其他维度也放进来。这样，这个额高基数的维度就
被隔离在一个聚合组肿了。

* Mandatory Dimensions(强制维度): 指的是那些总是会出现在where条件或group by 语句里的维度。通过制定某个维度为强制维度，`kylin`
可以不预计算那些不包含此维度的cube，从而减少计算量.
* Hierarchy Dimensions(层级维度): 指的是一组具有层级关系的维度，如省市县。当查询低级维度时，往往会带上高级维度的数据。通过指定
层级维度，`kylin`可以略过不满足次模式的Cuboid 
* Joint Dimensions(联合维度): 是将多个维度视作一个维度，进行组合计算的时候，它们要么一起出现，要么均不出现。通常适用于以下几种情形:
1. 总是一起查询的维度 2. 彼此之间又一定映射关系，如USER_ID和EMAIL 3. 基数很低的维度，如性别、布尔类型的属性。

这里还需要说一点。一个维度如果出现在某个属性中后，将不能再设置另一种属性。但是一个维度，可以出现在多个聚合组中。

下面说一说`rowkey`
* ![cube_6](https://pic.kuaizhan.com/g3/cf/f2/8fca-be61-4b29-9ef1-25f4970107b567)
 
`kylin`以key-value的形式将cube的构建结果存储到`hbase`中。HBase的RowKey，即行键是用来存储其他列的唯一索引。`kylin`需要按照多个
维度来对度量进行检索，因此存储到HBase的时候，需要将多个维度值拼接组成RowKey。
由于同一维度中的数值长短不一，所以就需要对维度值进行编码。
`kylin`目前支持的编码方式有以下几种:
* Dictionary编码: 字典编码将所有此维度下的值构建成一张映射表，从而大大节约存储空间。