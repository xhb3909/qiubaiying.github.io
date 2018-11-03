---
layout:     post
title:      keep
subtitle:   keep工作记录笔记
date:       2018-08-07
author:     haobo
header-img: img/post-bg-mma-3.jpg
catalog: true
tags:
    - keep
---


# ssh-add 失效

eval `ssh-agent -s` 使用这个命令激活ssh-add
ssh-add -K ~/keep/pem/xuhaobo.pem 将你的公钥添加到ssh-agent和Keychain中;
ssh-add -l 查看是否添加成功

```java

public class WebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) throws ServletException {

        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("dispatcher-servlet.xml");
        ServletRegistration.Dynamic dispatcher = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/*");

        // Filter
        FilterRegistration.Dynamic encodingFilter = container.addFilter("encodingFilter", CharacterEncodingFilter.class);
        encodingFilter.addMappingForUrlPatterns(null, false, "/*");
        encodingFilter.setInitParameter("encoding", "UTF-8");
        encodingFilter.setInitParameter("forceEncoding", "true");

        FilterRegistration.Dynamic corsFilter = container.addFilter("CORS", CORSFilter.class);
        corsFilter.addMappingForUrlPatterns(null, false, "/*");
        corsFilter.setInitParameter("allowUrl", "*");
        corsFilter.setInitParameter("cors.allowed.methods", "GET,POST,HEAD,OPTIONS,PUT");

        // Listener
        container.addListener(ContextLoaderListener.class);
        container.addListener(IntrospectorCleanupListener.class);
        container.addListener(RequestContextListener.class);

    }
}

```

# 删除远端分支

查看远端分支 git branch -r
删除远端分支 
git branch -r -d origin/branch-name
git push origin :branch-name

# 迁移spring boot 项目的问题汇总
1、buffet 项目模块之间循环依赖问题：
    model、utils、core 之间互相依赖
    应该明确划分模块边界
    util 应该为公用模块、可以被 model 使用，util 中不应该包含 model
    model 为 core、dao、service、web 公用模块
    
2、spring 相关依赖统一由 spring-boot-parent 管理，需要把之前相关 spring 版本依赖移除

3、log4j 配置问题
springboot 默认使用 logback，需要自定义使用 log4j2
参照 https://blog.csdn.net/woniu211111/article/details/54347846


具体 XML 配置参照官方 DOC

4、分模块的配置文件需要合并为一个配置文件，放在 core 模块中
没有查到父子模块场景下配置文件的继承，只能先放到一个文件中

5、java.lang.NoClassDefFoundError: 
类似的错误都是依赖包版本问题，引用的 commons 包里面某些 jar 包版本过低，而在项目中如果对该包进行了包版本管理，则会出现这个问题
解决方案是保持版本一致或者去掉项目中的包管理


6. SpringBoot启动报：Caused by: java.lang.IllegalArgumentException: At least one JPA metamodel must be present!
原因：本质上是因为 springBoot 如果扫描到 jpa-starter 依赖
（包含 JpaRepositoriesAutoConfiguration ），会自动寻找 jpa 的 properties 配置

解决方案：
1、去除 jpa-starter 依赖
2、或者
在Application启动类上加
@EnableAutoConfiguration
(exclude = {
       JpaRepositoriesAutoConfiguration.class
})注解

7、hbaseConfiguration 的初始化 ERROR
hbase-site.xml 中 ${hbase.node} 类型占位符变量无法进行替换

原因：未查明，跟 xml 中属性替换机制有关系
解决方案：
替换为 HbaseTemplate config 方式


8、xml文件中报${} 变量在application.properties中找不到, 需要在Application类中
注入
@Bean
public static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
   return new PropertySourcesPlaceholderConfigurer();
}
以此来支持${}的方式


9、commons-cache 包中 Redis 版本冲突问题
如果 spring-Boot 使用了 2.0.3，默认的 spring-data-redis 版本为 2.0.3-RELEASE，而 commons 包中使用的是 1.5.7-REALEASE，有个类 Connection.setEx() 是不兼容的，会导致运行时 NoMethodError


10、对于非 web 类模块，应该配置 WebApplicationType 并实现 CommandLineRunner

```java
@SpringBootApplication
@ImportResource(value = {"classpath:applicationContext-kafka-consumer.xml"})
public class WorkerApplication implements CommandLineRunner {

		@AutoWired
		private ApplicationContext applicationContext;

		public static void main(String[] args) {
				new
				SpringApplication(WorkerApplication.class).web(false).run(args);
		}

		@Override
		public void run(String... args) throws Exception {
				Map<String, AbstractKafkaMessageListener> listenerMap =
				applicationContext.getBeansOfType(AbstractKafkaMessageListener.class);

				String brokers = applicationContext.getBean("kafkaBrokerHost",
				String.class);

				KafkaConsumerStarter kafkaConsumerStarter = new
				KafkaConsumerStart(brokers);

				kafkaConsumerStarter.start(listenerMap.values());
		
		}



}

```



11、
Description:

The Bean Validation API is on the classpath but no implementation could be found

Action:

Add an implementation, such as Hibernate Validator, to the classpath

在当前项目下添加
<dependency>
   <groupId>org.hibernate</groupId>
   <artifactId>hibernate-validator</artifactId>
</dependency>








