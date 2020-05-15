title: 利用Autoconfig打包Java WEB应用
date: 2015-10-17 14:39:08
tags: 
- java
- package 
- autoconfig
categories: 部署
description: 本文主要介绍常用的后台应用打包的几种方式
---

简介： 本文主要介绍常用的后台应用打包的几种方式

后端应用上线前都需要经过重新打包，可千万别小看了这个阶段，这个是非常非常重要的！如果打错了包或者使用错了配置文件，结果可能是毁灭性的！

我们都知道每个软件项目或者公司都会维护几套隔离环境，例如以前在阿里就会有`日常测试`、`日常`、`预发`和`线上`几个环境，还有根据特殊需要配置的独立环境，如`性能环境`等等。 当然，对于小公司或者创业公司来说不需要准备这么多套环境，但至少是需要`测试`和`线上`两套环境的。多套环境的可以有效的隔离线上和线下，提高开发人员的工作效率，又不至于将不稳定的代码带到线上。其中最重要的一个环节就是打包，我主要介绍两种简单的打包方式。

## 利用Spring配置

现在Java WEB应用可以说90%都使用了Spring框架，而Spring框架早就帮我们考虑了这个问题。我一开始也是使用这个配置方式，在Spring配置文件中引入一下配置：

> <context:property-placeholder location="file:${APP_HOME}/config.properties"/>

Spring是支持classpath和file的，个人推荐使用`file`模式来查找外部配置文件，因为这样我们就不必将配置文件引入到工程目录中了，因为工程目录对所有的开发人员都可见，这样会降低配置文件的安全性。引入外部配置文件一个常见的做法就是使用环境变量，我们新建一个`APP_HOME`的环境变量来区分不同的环境。

在使用配置文件的地方利用placeholder进行配置即可，例如以下方式：

```
<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
        <property name="driverClassName" value="${db.driverClass}"/>
        <property name="url" value="${db.url}"/>
        <property name="username" value="${db.username}"/>
        <property name="password" value="${db.password}"/>
</bean>
```
在Spring启动以后，它会去查找你配置的外部配置文件，并逐个替换使用的配置中的placeholder。

这种方式的优点就是简单，灵活，但是缺点也是很明显的：

* 只支持Spring配置文件的替换，不支持其他框架配置文件的替换。
   如果你想替换logback.xml中的某个配置，例如日志输出目录或者日志输出级别，它是做不到的。
   
* 大规模部署不方便。
  如果只有一两机器这样部署还是比较方便的，但是如果有几十台甚至上百台这样打包就十分麻烦了。如果改动一个配置项，就需要保持所有机器的同步的，所以一般大一点的公司都会有专门负责配置的服务，例如阿里的ConfigServer。
  
## 利用AutoConfig打包
  
AutoConfig 是阿里内部使用的一个打包工具，十分方便，也十分强大，这里有它的介绍：<http://openwebx.org/docs/autoconfig.html>

下面是我利用AutoConfig打包的步骤：

* 添加不同环境的配置

  为了直接利用Maven打出不同环境的包，我们在需要打包的module的pom中添加下面的配置：
  
  ```
  <properties>
        <autoconfig.properties>antx.properties.dev</autoconfig.properties>
        <env>dev</env>
  </properties>
  ```
然后加入profile配置：

  ```
 <profiles>
        <profile>
            <!-- 本地开发环境 -->
            <id>dev</id>
            <properties>
                <autoconfig.properties>antx.properties.dev</autoconfig.properties>
                <env>dev</env>
            </properties>
        </profile>
        <profile>
            <!-- 测试环境 -->
            <id>test</id>
            <properties>
                <autoconfig.properties>antx.properties.test</autoconfig.properties>
                <env>test</env>
            </properties>
        </profile>
        <profile>
            <!-- 生产环境 -->
            <id>online</id>
            <properties>
                <autoconfig.properties>antx.properties.online</autoconfig.properties>
                <env>online</env>
            </properties>
        </profile>
    </profiles>
    ```
    
 * 添加autoconfig maven插件支持
  
 ```
 <plugin>
	<groupId>com.alibaba.citrus.tool</groupId>
          <artifactId>autoconfig-maven-plugin</artifactId>
 	<version>1.2</version>
	<configuration>
               	<userProperties>${user.home}/conf/${autoconfig.properties}</userProperties>
	</configuration>
	<executions>
                    <execution>
                        <goals>
                            <goal>autoconfig</goal>
                        </goals>
                    </execution>
	</executions>
</plugin>
```
 
 其中,userProperties属性就是我们使用的配置文件。
 
 * 利用Maven进行打包
   
   进入到需要打包的module中，然后执行`mvn package -P<env>`，其中env代表不同的环境，在上面的配置中env只能为dev、test和online.
   我们可以将最终的包名也带上环境名称，以区分打出来的不同环境的包，如下配置：
   
`<finalName>包名-${env}</finalName>`


### Tips:

如果开发人员使用的是jetty插件来进行本地开发的，那么需要将jetty:run改为jetty:run-war，因为autoconfig是需要执行package才会进行触发的，而jetty:run不会执行package阶段。可以参考一下配置：

    ```
           <!-- jetty插件 -->
            <plugin>
                <groupId>org.mortbay.jetty</groupId>
                <artifactId>jetty-maven-plugin</artifactId>
                <version>7.6.16.v20140903</version>
                <configuration>
                    <webAppSourceDirectory>src/main/webapp</webAppSourceDirectory>
                    <scanIntervalSeconds>3</scanIntervalSeconds>
                    <connectors>
                        <connector implementation="org.eclipse.jetty.server.nio.SelectChannelConnector">
                            <port>8080</port>
                            <maxIdleTime>60000</maxIdleTime>
                        </connector>
                    </connectors>
                    <war>target/包名-${env}.war</war>
                </configuration>
            </plugin>
```

