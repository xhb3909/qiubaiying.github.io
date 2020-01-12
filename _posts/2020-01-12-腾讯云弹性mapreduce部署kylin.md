---
layout:     post
title:      腾讯云弹性mapreduce部署kylin
date:       2020-01-10
author:     xhb
header-img: img/post-bg-iWatch.jpg
catalog: true
sid: 20200112
tags:
    - kylin
---

# 开篇
公司最近想要重构之前的统计系统，想要能够通过多个维度聚合查询出来不同的指标。之前的一个做法是sparkStreaming实时流处理把统计日志信息清洗，
最后保存到hive中。离线任务通过spark on hive sql，聚合hive数据，把单个维度对应的指标数据保存到mongo里。这样的做法，显然不满足提出的根据
多个维度聚合查询的需求。
有同学可能会问了，我们可以每次查询的时候，都调用hive sql聚合数据返回给用户不就好了吗? 
但是这样的做法不好。
第一方面: 每次调用hive sql都相当于在yarn上执行多次mapReduce任务，当每秒查询数据条数增加，无疑对集群来说是一种压力。
第二方面: 调用hive sql, 返回数据的时间也难以让人接受，千万级的数据多维聚合都要7-8秒才能返回。
所以基于这两点考虑用`kylin`来实现我们公司的需求。对于`kylin`不了解的同学，可以看我写之前写的文章:
[kylin初出茅庐](https://xhb3909.com/2020/01/10/kylin%E5%88%9D%E5%87%BA%E8%8C%85%E5%BA%90/)

# 腾讯云弹性mapReduce搭建`kylin`
因为公司现在的大数据组件，都是在腾讯云弹性mapReduce上搭建的。所以这次也是在腾讯云emr基础上搭建`kylin`。
我们可以先看一下在腾讯云上emr的配置:
```
hadoop-2.7.3,
spark_hadoop2.7-2.0.2,
zookeeper-3.4.9,
hbase-1.2.4,
hive-2.1.1

```
那么`kylin`对于其他组件有什么要求呢?
```
Hadoop: 2.7+, 3.1+ (since v2.5)
Hive: 0.13 - 1.2.1+
HBase: 1.1+, 2.0 (since v2.5)
Spark (可选) 2.3.0+
Kafka (可选) 1.0.0+ (since v2.5)
JDK: 1.8+ (since v2.5)
OS: Linux only, CentOS 6.5+ or Ubuntu 16.0.4+

```

通过对比，我们发现Hadoop、Hive、HBase满足要求，Spark没有满足要求，版本偏低，所以我们这次搭建不使用spark做计算框架，使用Hadoop自带的MapReduce
来实现计算。

1. 我们可以从官网下载`apache-kylin-3.0.0-bin-hbase1x.tar.gz`解压缩

```shell

wget -b http://mirrors.tuna.tsinghua.edu.cn/apache/kylin/apache-kylin-3.0.0/apache-kylin-3.0.0-bin-hbase1x.tar.gz

```

2. 解压缩成功之后，需要修改`kylin/conf`目录下的`setenv.sh`文件

```
export KYLIN_JVM_SETTINGS="-Xms4g -Xmx8g -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:$KYLIN_HOME/logs/kylin.gc.%p 
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=64M"

```

3. 配置`kylin/conf`目录下的`kylin.properties`文件

```
kylin.env.zookeeper-base-path=/kylin
kylin.env=QA
kylin.env.hdfs-working-dir=/kylin

```

4. tomcat去掉https相关配置，去掉`kylin/tomcat/conf`目录下的server.xml文件内容如下所示:

```
        <!-- <Connector port="7443" protocol="org.apache.coyote.http11.Http11Protocol"
                   maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
                   keystoreFile="conf/.keystore" keystorePass="changeit"
                   clientAuth="false" sslProtocol="TLS" /> -->

```

5. 把`hbase-site.xml`、`hive-site.xml`、`core-site.xml`拷贝到`kylin/conf`目录下

6. 配置`hive-site.xml`

```
<property>
        <name>metastore.storage.schema.reader.impl</name>
        <value>org.apache.hadoop.hive.metastore.SerDeStorageSchemaReader</value>
</property>

```

7. `hive/lib`下的包拷贝到`kylin/lib`目录下
8. `hbase/lib`下的包拷贝到`kylin/lib`
9. `hive/hcatalog/share/hcatelog`包拷贝到`kylin/lib`
10. 最关键的一步，需要在/etc/profile里配置KYLIN_HOME

```
export KYLIN_HOME=/usr/local/service/kylin
PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HIVE_HOME/bin:$HBASE_HOME/bin:$SPARK_HOME/bin:$KYLIN_HOME/bin:$PATH

```

经过以上几步的部署，我们就可以通过命令来启动`kylin`实例

运行`$KYLIN_HOME/bin/kylin.sh start`界面输出如下:

```

Retrieving hadoop conf dir...
KYLIN_HOME is set to /usr/local/apache-kylin-2.5.0-bin-hbase1x
......
A new Kylin instance is started by root. To stop it, run 'kylin.sh stop'
Check the log at /usr/local/apache-kylin-2.5.0-bin-hbase1x/logs/kylin.log
Web UI is at http://<hostname>:7070/kylin

```

使用`kylin`:

`Kylin` 启动后您可以通过浏览器 http://hostname:7070/kylin 进行访问。其中 hostname为具体的机器名、IP 地址或域名，默认端口为 7070。
初始用户名和密码是 ADMIN/KYLIN。服务器启动后，您可以通过查看 $KYLIN_HOME/logs/kylin.log 获得运行时日志。

访问界面如图所示:
![界面](https://pic.kuaizhan.com/g3/cb/d1/5838-c5d1-48bf-a493-431923bac22237)

登录之后就可以看到如下界面:
![登录之后](https://pic.kuaizhan.com/g3/00/b7/5b79-d73c-4a16-8acc-eeaa35a75cf906)

# 总结
这样我们就完成了在emr集群搭建`kylin`，之后的博客我会分享如何在`kylin`上配置model和cube，有哪些调优的点需要注意。感谢大家的观看。