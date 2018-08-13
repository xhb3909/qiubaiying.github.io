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
