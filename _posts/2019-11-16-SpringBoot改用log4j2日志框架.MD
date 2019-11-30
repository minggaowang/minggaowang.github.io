---
layout:     post
title:      SpringBoot改用log4j2日志框架
subtitle:   log4j2
date:       2019-11-16
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - SpringBoot
---

## **log4j2日志框架介绍** 

log4j2是log4j 1.x和logback的改进版，据说采用了一些新技术（无锁异步、等等）,使得日志的吞吐量、性能比log4j 1.x提高10倍，并解决了一些死锁的bug,而且配置更加简单灵活。 

##  **log4j2日志级别**

 在log4j2中, 共有8个级别,按照从低到高为：ALL < TRACE < DEBUG < INFO < WARN < ERROR < FATAL < OFF。 

- All:最低等级的，用于打开所有日志记录。
- Trace:是追踪，就是程序推进一下。
- Debug:指出细粒度信息事件对调试应用程序是非常有帮助的。
- Info:消息在粗粒度级别上突出强调应用程序的运行过程。
- Warn:输出警告及warn以下级别的日志。
- Error:输出错误信息日志。
- Fatal:输出每个严重的错误事件将会导致应用程序的退出的日志。
- OFF:最高等级的，用于关闭所有日志记录。

##  **配置pom.xml文件** 

 其中要把spring-boot-starter和spring-boot-starter-web的依赖包排除logging的引用，这个默认是引入logback日志框架。还有要把spring-boot-starter-log4j2依赖包也引用进来。

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter</artifactId>
	<exclusions>
		<exclusion>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-logging</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

##  **配置application-prod.yml配置**

```yaml
logging:
 config: classpath:log4j2.xml
```

##  **log4j2配置文件log4j2.xml** 

 以mybatis打出sql日志为例子配置： 

```xml
<Configuration status="off">
<Properties>
	<Property name="LOG_PATH">.</Property>
</Properties>
<Appenders>
	<!-- Console -->
	<Console name="Console" target="SYSTEM_OUT">
		<PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %logger{36}.%M@%L - %msg%n"/>
		<!--<PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %C{5} (%F:%L) - %msg%n"/>-->
	</Console>
	<!-- RollingFile -->
	<RollingRandomAccessFile name="RollingFile"
							 fileName="./log/first-app-by-gui.log"
							 filePattern="${LOG_PATH}/first-app-by-gui.log.%d{yyyy-MM-dd-HH}">
		<PatternLayout pattern="%date{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger{36}.%M@%L - %msg%n"/>
		<Policies>
			<TimeBasedTriggeringPolicy/>
		</Policies>
		<DefaultRolloverStrategy max="30"/>
	</RollingRandomAccessFile>
</Appenders>

<Loggers>
	<!-- Spring Loggers -->
	<Logger name="org.springframework" level="warn"/>

	<!-- mainbootAdmin Loggers -->
	<Logger name="springboot.firstappbygui" level="info" additivity="false">
		<AppenderRef ref="Console"/>
		<AppenderRef ref="RollingFile"/>
	</Logger>
	
	
	<logger name="springboot.firstappbygui.mapper" level="DEBUG" additivity="false">
		 <AppenderRef ref="Console"/>
		 <AppenderRef ref="RollingFile"/>
	</logger>

	<!-- Root Logger -->
	<Root level="warn">
		<AppenderRef ref="Console"/>
		<AppenderRef ref="RollingFile"/>
	</Root >
</Loggers>
</Configuration>
```

###  log4j2.xml配置说明 

####  **根节点Configuration** 

 有两个属性和两个子节点： 

- 属性status:status用来指定log4j本身的打印日志的级别。
- 属性monitorinterval：monitorinterval用于指定log4j自动重新配置的监测间隔时间，单位是s,最小是5s。
- 节点Appenders和Loggers(表明可以定义多个Appender和Logger)。

####  **节点Appenders** 

 常见的有三种子节点:Console、RollingFile、File。 

 Console节点用来定义输出到控制台的Appender: 

- name:指定Appender的名字。
- target:SYSTEM_OUT 或 SYSTEM_ERR,一般只设置默认:SYSTEM_OUT。
- PatternLayout:输出格式，不设置默认为:%m%n。

 File节点用来定义输出到指定位置的文件的Appender： 

- name:指定Appender的名字。
- fileName:指定输出日志的目的文件带全路径的文件名。
- PatternLayout:输出格式，不设置默认为:%m%n。

 RollingFile节点用来定义超过指定大小自动删除旧的创建新的的Appender： 

- name:指定Appender的名字。
- fileName:指定输出日志的目的文件带全路径的文件名。
- PatternLayout:输出格式，不设置默认为:%m%n。
- filePattern:指定新建日志文件的名称格式。
- Policies:指定滚动日志的策略，就是什么时候进行新建日志文件输出日志。
- TimeBasedTriggeringPolicy:Policies子节点，基于时间的滚动策略，interval属性用来指定多久滚动一次，默认是1 hour。modulate=true用来调整时间：比如现在是早上3am，interval是4，那么第一次滚动是在4am，接着是8am，12am...而不是7am。
- SizeBasedTriggeringPolicy:Policies子节点，基于指定文件大小的滚动策略，size属性用来定义每个日志文件的大小。
- DefaultRolloverStrategy:用来指定同一个文件夹下最多有几个日志文件时开始删除最旧的，创建新的(通过max属性)。

####  **子节点Loggers（常见的有两种:Root和Logger）** 

 Root节点用来指定项目的根日志，如果没有单独指定Logger，那么就会默认使用该Root日志输出： 

- level:日志输出级别，共有8个级别，按照从低到高为：All < Trace < Debug < Info < Warn < Error < Fatal < OFF。
- AppenderRef：Root的子节点，用来指定该日志输出到哪个Appender。

 Logger节点用来单独指定日志的形式，比如要为指定包下的class指定不同的日志级别等: 

- level:日志输出级别，共有8个级别，按照从低到高为：All < Trace < Debug < Info < Warn < Error < Fatal < OFF。
- name:用来指定该Logger所适用的类或者类所在的包全路径,继承自Root节点。
- AppenderRef：Logger的子节点，用来指定该日志输出到哪个Appender,如果没有指定，就会默认继承自Root.如果指定了，那么会在指定的这个Appender和Root的Appender中都会输出，此时我们可以设置Logger的additivity="false"只在自定义的Appender中进行输出。

###  log4j调优和注意事项 

####  **日志模式-同步/异步** 

##### 具体描述说明

 log4j2提供了AsyncAppender和AsyncLogger以及全局异步，开启方式如下: 

- 同步模式：默认配置即为同步模式，即没有使用任何AsyncAppender和AsyncLogger。

  ![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/676975-20190827212511565-1949597663.png)

- 全局异步：配置按照同步方式配，通过添加jvm启动参数即可开启全局异步，无需修改配置和应用。

  ![]( https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/676975-20190827212857292-1887643935.png )

- 混合异步：使用异步Logger和同步Logger的混合配置，且不开启全局异步，即Logger配置中部分AsyncLogger，部分Logger。

##### 同步/异步日志输出方式

-  Sync： 同步打印日志，日志输出与业务逻辑在同一线程内，当日志输出完毕，才能进行后续业务逻辑操作 。
-  Async Appender： 异步打印日志，内部采用ArrayBlockingQueue，对每个AsyncAppender创建一个线程用于处理日志输出。 
-  Async Logger ： 异步打印日志，采用了高性能并发框架Disruptor，创建一个线程用于处理日志输出。 

#####  使用注意事项

- 如果使用异步，建议使用AsyncLogger实现而不是AsyncAppender。
- 如果使用同步，AsyncLogger、AsyncAppender和全局异步只能使用一种，不可以同时配置AsyncAppender和AsyncLogger，或者配置了异步的情况下启用了全局异步。

####  **日志滚动和清除策略** 

 log4j2提供了基于文件大小的滚动策略和基于时间的滚动策略，也可以二者并用，这里给出基于大小的滚动策略配置和基于大小/时间双滚动策略配置: 

- 基于大小的滚动策略:按照大小滚动，启用压缩，并最多保留N个文件
- 基于大小/时间双滚动滚动策略:按照大小和时间滚动，启用压缩，单位时间内控制最多保留日志个数并控制总的日志留存时间。
