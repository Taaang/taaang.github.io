---
title: MySQL Communications link failure问题
date: 2016-12-10 11:34:29
categories:
- Common
tags:
- MySQL
---


最近对项目进行测试，突然出现Communications link failure异常，原文如下：

```
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 20,096 milliseconds ago.The last packet sent successfully to the server was 0 milliseconds ago.
    at sun.reflect.GeneratedConstructorAccessor84.newInstance(Unknown Source) ~[?:?]
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[?:1.7.0_75]
    ...
```

查看MySQL Status的Aborted_clients，变化如下：
请求接口前：

![mysql_1](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_1.jpeg?raw=true)
![mysql_2](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_2.jpeg?raw=true)

请求接口后：

![mysql_3](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_3.jpeg?raw=true)
![mysql_4](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_4.jpeg?raw=true)

等待一段时间后：

![mysql_5](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_5.jpeg?raw=true)
![mysql_6](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_6.jpeg?raw=true)

此时，再次请求接口，出现Communications link failure，可判断是MySQL认为连接已经无效，主动关闭了连接。查测试服务器MySQL相关配置后，发现wait_timeout设置为10s，再结合日志中last packet successfully received时间，判断可能为该配置问题。

但是比较奇怪的是，线上项目正常。正常情况下，线上服务器应与测试服务器上的配置一致，保证测试结果有效。所以果断查看线上MySQL配置后，发现wait_timeout为600s，所以基本确定是因为测试服务器wait_timeout最近可能被人修改，导致出现这个MySQL主动关闭连接的情况。

这又引出另一个问题，如果线上配置是600s，那么有过期连接后，MySQL依然会进行关闭，应该也会出现同样的错误，但是实际情况是线上项目一直正常，没有出现告警，于是怀疑是获取连接的时候，连接池进行了一些维护性工作。

连接池使用的是阿里的DruidDataSource，查看连接其默认配置，发现：

![mysql_druid](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_status_3.jpeg?raw=true)

其中testOnBorrow和testOnReturn是默认不开启的，testWhileIdle默认为true，查看获取连接的相关代码，发现如下：

```
if (isTestWhileIdle()) {
    final long currentTimeMillis = System.currentTimeMillis();
    final long lastActiveTimeMillis = poolableConnection.getConnectionHolder().getLastActiveTimeMillis();
    final long idleMillis = currentTimeMillis - lastActiveTimeMillis;
    long timeBetweenEvictionRunsMillis = this.getTimeBetweenEvictionRunsMillis();
    if (timeBetweenEvictionRunsMillis <= 0) {
        timeBetweenEvictionRunsMillis = DEFAULT_TIME_BETWEEN_EVICTION_RUNS_MILLIS;
    }

    if (idleMillis >= timeBetweenEvictionRunsMillis) {
        boolean validate = testConnectionInternal(poolableConnection.getConnection());
        if (!validate) {
            if (LOG.isDebugEnabled()) {
            LOG.debug("skip not validate connection.");
            }
            discardConnection(realConnection);             
            continue;
        }
    }
}
```

其中，默认的DEFAULT_TIME_BETWEEN_EVICTION_RUNS_MILLIS为60s，即默认对超过60s的未活动连接进行检测，所以线上MySQL配置600s也没有出现Communications link failure。
