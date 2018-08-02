---
title: Tomcat类加载NoSuchMethodError异常问题
date: 2018-07-13 22:15:36
categories:
- JVM
tags:
- JVM
- NoSuchMethodError
---

最近在开发环境中进行项目部署测试，再次遇一个鬼故事。项目maven打包正常，Tomcat上项目正常启动，但是在调用某个类的特定方法时，出现NoSuchMethodError异常，反编确认对应的JAR包中包含该类和对应的方法，但是JVM依旧报错。

## 异常信息

```
[ERROR] 15:35:43.751 c.c.m.o.e.GlobalExceptionHandler(31) - Handler dispatch failed; nested exception is java.lang.NoSuchMethodError: com.google.common.base.Splitter.splitToList(Ljava/lang/CharSequence;)Ljava/util/List;
org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.NoSuchMethodError: com.google.common.base.Splitter.splitToList(Ljava/lang/CharSequence;)Ljava/util/List;
        at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:978)
        at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:897)
        at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
        at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:872)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:644)
        at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:725)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:291)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
        at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:239)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:206)
```

## 异常分析

项目是在尝试调用Google Guava Splitter中的splitToList方法时，发现该方法未定义。

首先，项目编译时正常，所以在此过程中，是能找到对应的类和方法，那么为什么会出现在运行时找不到的情况呢？

其次，抛出的异常是NoSuchMethodError，说明ClassLoader成功加载到了对应的类，只是在进行方法调用时，发现方法不存在。

针对上面的分析，进行一些猜想的验证：

&emsp;1.maven打包使用了低版本的Guava，而低版本中没有对应的方法

&emsp;理论上来说，这种情况不太可能，如果使用了没有对应方法的低版本，那么打包编译是会失败的，实际上反编后，也是可以在对应的类中找到该方法的；

&emsp;2.JAR冲突，导致使用了低版本的Guava，而低版本中没有对应的方法

&emsp;然鹅，事实证明不是，maven依赖树中只出现了一个guava包的引用，所以排除这种情况；

&emsp;3.Tomcat进行Class加载时，加载了低版本的Guava，而低版本中没有对应的方法

&emsp;于是，我把项目打成了JAR，引入tomcat模块，放到测试环境中运行，一切正常~那么基本可以判断是和服务器Tomcat或JAVA环境有一定的关系，为了弄清楚在加载该类时，是从哪里加载的，在Tomcat启动参数中，加入-XX:+TraceClassLoading -XX:+TraceClassUnloading，用于跟踪Tomcat类加载和卸载，并在日志中进行查看，恩。。。然后发现：

![jvm_load_error](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_jvm_class_load.png?raw=true)

恩。。。。。。。

恩。。。。。。。。

恩。。。。。。。。。

嘛玩意儿这是。。。。。

![jvm_load_error_gg](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_jvm_class_load_gg.jpg?raw=true)

根本就是不是从我的JAR包中加载的。。。。

从上面的图可以看出，在进行Guava Splitter类加载的时候，从JRE的扩展LIB库中进行了该类的加载，那么为什么没有从项目包中加载，而是从扩展LIB库中进行加载的呢？

可以看出，只有在项目部署在服务器Tomcat上的时候，才出现了相应的问题，那么很可能是和服务器中的Tomcat或JAVA环境有关。

首先，我们来看一下Tomcat是如何做类加载和管理的。

(https://tomcat.apache.org/tomcat-8.0-doc/class-loader-howto.html)

Tomcat使用一系列不同的ClassLoader来加载和管理一些基础的常规类，它们是能被所有WEB应用同时使用的，包括JVM的基础运行时类、Tomcat内部类等。而单个应用部署在Tomcat的独立容器中，每个容器中的WEB应用对应的ClassLoader相互隔离，以此来实现隔离。Tomcat中ClassLoader之间是呈现层级关系的，具体结构如下图所示：

![tomcat_class_loader_structure](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_tomcat_class_loader_structure.png?raw=true)

其中，

__Bootstrap class loader__  用于加载JVM运行时基础类，同时也包含了扩展JAR包目录下的所有类（$JAVA_HOME/jre/lib/ext）；

__System class loader__  一般从CLASSPATH环境变量中进行类初始化，所有的这些对于Tomcat内部类和WEB应用都是可见的。然后，标准的Tomcat启动脚本是会忽略CLASSPATH环境变量，而使用下列仓库进行代替：

&emsp;1.$CATALINA_HOME/bin/bootstrap.jar  包含用于初始化Tomcat的所需类；

&emsp;2.$CATALINA_BASE/bin/tomcat-juli.jar or $CATALINA_HOME/bin/tomcat-juli.jar  日志相关的实现类；

&emsp;3.$CATALINA_HOME/bin/commons-daemon.jar  Apache Commons Daemon 中的类，Linux下可用于实现后台服务，Windows下可用于实现注册为系统服务。

__Common class loader__  包含附加的类，对于Tomcat内部类和所有的WEB应用都是可见的，默认查找CATALINA_HOME和CATALINA_BASE目录下的lib目录；

__WebappX class loader__  为每一个部署在Tomcat中WEB应用实例创建的ClassLoader，包括/WEB-INF/classes下所有解包的类和资源，同时也包含/WEB-INF/lib目录下JAR包中的类和资源。

由于前三个ClassLoader是各个WEB应用通用的，当需要加载一个类时，会优先按顺序上最上层ClassLoader进行加载，当一个ClassLoader在对应的目录中没有找到对应的类时，就依次交给下一层级的ClassLoader进行加载。如果最终都没有找到对应的类，则抛出NoClassDefFoundError。

综上所述，当尝试进行Google Guava Splitter类的加载时，首先交由Bootstrap class loader进行尝试性的加载，然后在对应目录中的JAR包中进行搜索时，居然从jre/lib/ext目录下的JAR包中找到了对应的类，所以优先进行了加载，而在对该JAR包进行反编后，发现确实包含了Google Guava，并且对应的Guava版本是14.0.1，而在对应的版本下，Guava Splitter确实还没有splitToList方法，所以最终导致类加载成功，但是找不到对应的方法，抛出NoSuchMethodError。

## 异常原因

当尝试进行Google Guava Splitter类的加载时，Bootstrap class loader优先找到并加载了低版本的Guava Splitter，而低版本中没有包含splitToList方法。

后来在同事里问了一圈，发现是有同事之前测试，将测试JAR包放到了jre/lib/ext目录，导致了该问题的发生。

## 解决方案

1. 经确认后，将该JAR从扩展库目录中移除，处理后恢复正常；

2. 禁止将业务JAR包放到JAVA库目录中，避免相关问题的再次发生；

3. 在对模块进行JAR包封装，或进行SDK开发时，应尽可能较少相关第三方库依赖，避免与引用方依赖发生冲突；

4. 如果无法避免引用第三方常用类库，可以使用maven shade插件，对相关第三方类库的包名进行变更，以此避免相关问题。
