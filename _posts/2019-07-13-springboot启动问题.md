---
layout:     post
title:      spring boot jackson bind exception 
subtitle:   spring boot 整合spark、hive、出现的问题 
date:       2019-07-13 
author:     haobo
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - springboot 
---


# spring boot 整合spark、hive、hbase报错

## maven 相关配置如下

### 父pom.xml文件
```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/>
    </parent>

	<dependencies>
		<dependency>
		   <groupId>com.fasterxml.jackson.core</groupId>
           <artifactId>jackson-databind</artifactId>
           <version>2.9.0</version>	
		</dependency>
	</dependencies>
```

### 子pom直接引用父pom关于jackson-databind的依赖

```
项目运行起来报错:

org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'formContentFilter' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.boot.web.servlet.filter.OrderedFormContentFilter]: Factory method 'formContentFilter' threw exception; nested exception is java.lang.NoClassDefFoundError: Could not initialize class com.fasterxml.jackson.databind.ObjectMapper
```

#### 分析错误原因

```
BeanCreationException 和 'formContentFilter'
stackoverflow 搜索解决方案
https://stackoverflow.com/questions/43701983/spring-boot-cant-start-with-embedded-tomcat-error/43702686

参考里面的配置方式，解决问题
```
