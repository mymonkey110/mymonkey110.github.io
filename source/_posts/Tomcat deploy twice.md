title: 'Debug:Tomcat deploy twice'
date: 2015-10-30 10:46:31
tags: 
- tomcat
- 部署
- debug
categories: 踩过的那些坑
description: tomcat重复部署的问题
---

最近写了一个分布式时间调度系统，用于调度集群类的定时任务程序。架构如下：

![](http://7xnwpq.com1.z0.glb.clouddn.com/jscheduler.png)

有一个集中化的Scheduler来调度集群中所有的job,集群中的job只负责实现具体job内容，而Trigger的定义和管理均在Scheduler中实现。Trigger通过MQ将触发消息发送到集群中的某台机器上。

在部署Scheduler的过程中观察日志如下出现以下奇怪的现象：

![debug-1](http://7xnwpq.com1.z0.glb.clouddn.com/E77611B1-937A-49AD-B51D-797143F9B6B1.png)

我们发现在同一时刻Scheduler对一个Job触发了两次，而在集群的某台机器上发现一个Job被触发了4次：

![debug-2](http://7xnwpq.com1.z0.glb.clouddn.com/9CDEE719-9454-4E4B-99C3-684518E9F49F.png)

当我在自己的机器上始终无法复现该问题。由于是同一个war包，故排出了代码的问题。不同之处在于，我本机启动的方式是用jetty的插件直接启动的，而服务器上则是用的是tomcat容器。经过一番排查发现，是tomcat重复部署的问题，tomcat的[官方文档]有如下说明:

>Any Context Descriptors will be deployed first.

[官方文档]:https://tomcat.apache.org/tomcat-7.0-doc/deployer-howto.html

因为我想讲应用直接部署在/下，所以在server.xml中的localhost节点下加入了context的配置。根据tomcat的官方文档，如果host下面有context的配置则会先部署，而后容器再部署一次。也就是说，应用被部署了两次。这也就解释了为什么scheduler会触发两次，而job会触发4次了。

解决的方法是将`deployOnStart`设置为`false`，`autoDeploy`设置为`false`。

参考：

<http://stackoverflow.com/questions/7223108/quartz-job-runs-twice-when-deployed-on-tomcat-6-ubuntu-10-04lts>