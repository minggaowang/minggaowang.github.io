---
layout:     post
title:      分布式搜索引擎ElasticSearch
subtitle:   ElasticSearch
date:       2019-11-30
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - ElasticSearch
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/home-banner-tencent-alibaba-cloud.png)

## 1 ElasticSearch简介

### **1.1** 什么是**ElasticSearch** 

​      Elasticsearch是一个实时的分布式搜索和分析引擎。它可以帮助你用前所未有的速 度去处理大规模数据。ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分 布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发 的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用 于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

### **1.2 ElasticSearch**特点

- 可以作为一个大型分布式集群（数百台服务器）技术，处理PB级数据，服务大公 司；也可以运行在单机上 
- 将全文检索、数据分析以及分布式技术，合并在了一起，才形成了独一无二的ES； 
- 开箱即用的，部署简单 
- 全文检索，同义词处理，相关度排名，复杂数据分析，海量数据的近实时处理

### **1.3 ElasticSearch**体系结构

下表是Elasticsearch与MySQL数据库逻辑结构概念的对比 

| Elasticsearch  | 关系型数据库Mysql |
| -------------- | ----------------- |
| 索引(index)    | 数据库(databases) |
| 类型(type)     | 表(table)         |
| 文档(document) | 行(row)           |

## **2 ElasticSearch**部署与启动 

下载ElasticSearch 5.6.8版本 

https://www.elastic.co/downloads/past-releases/elasticsearch-5-6-8

无需安装，解压安装包后即可使用 

在命令提示符下，进入ElasticSearch安装目录下的bin目录,执行命令 

```shell
elasticsearch
```

即可启动

我们打开浏览器，在地址栏输入http://127.0.0.1:9200/ 即可看到输出结果

```json
{
    "name":"uV2glMR",
    "cluster_name":"elasticsearch",
    "cluster_uuid":"RdV7UTQZT1‐Jnka9dDPsFg",
    "version":{
        "number":"5.6.8",
        "build_hash":"688ecce",
        "build_date":"2018‐02‐16T16:46:30.010Z",
        "build_snapshot":false,
        "lucene_version":"6.6.1"
    },
    "tagline":"You Know, for Search"
}
```

## **3 Head**插件的安装与使用 

### **3.1 Head**插件安装 

如果都是通过rest请求的方式使用Elasticsearch，未免太过麻烦，而且也不够人性化。我们一般都会使用图形化界面来实现Elasticsearch的日常管理，最常用的就是Head插件

步骤1： 

下载head插件：https://github.com/mobz/elasticsearch-head

步骤2：

解压到任意目录，但是要和elasticsearch的安装目录区别开。 

步骤3： 

安装node js ,安装cnpm 

```shell
npm install ‐g cnpm ‐‐registry=https://registry.npm.taobao.org
```

步骤4： 

将grunt安装为全局命令 。Grunt是基于Node.js的项目构建工具。它可以自动运行你所 设定的任务

```shell
npm install ‐g grunt‐cli
```

步骤5：安装依赖 

```shell
cnpm install
```

步骤6：

进入head目录启动head，在命令提示符下输入命令 

```
grunt server
```

步骤7： 

打开浏览器，输入 http://localhost:9100

步骤8： 

点击连接按钮没有任何相应，按F12发现有如下错误

No 'Access-Control-Allow-Origin' header is present on the requested resource

这个错误是由于elasticsearch默认不允许跨域调用，而elasticsearch-head是属于前端工 程，所以报错。

我们这时需要修改elasticsearch的配置，让其允许跨域访问。 

修改elasticsearch配置文件：elasticsearch.yml，增加以下两句命令：

```yml
http.cors.enabled: true 
http.cors.allow‐origin: "*"
```

此步为允许elasticsearch跨越访问 点击连接即可看到相关信息

### **3.2 Head**插件操作

#### **3.2.1** 新建索引 

选择“索引”选项卡，点击“新建索引”按钮

#### **3.2.2** 新建或修改文档 

在复合查询中提交地址，输入内容，提交方式为PUT

#### **3.2.3** 搜索文档 

选择"数据浏览"选项卡

#### **3.2.4** 删除文档 

在复合查询中提交地址，输入内容，提交方式为Delete

## **4 IK**分词器

### 4.1 什么是IK分词器

我们在浏览器地址栏输入http://127.0.0.1:9200/_analyze? analyzer=chinese&pretty=true&text=我是程序员，浏览器显示效果如下

```json
{
    "tokens":[
        {
            "token":"我",
            "start_offset":0,
            "end_offset":1,
            "type":"<IDEOGRAPHIC>",
            "position":0
        },
        {
            "token":"是",
            "start_offset":1,
            "end_offset":2,
            "type":"<IDEOGRAPHIC>",
            "position":1
        },
        {
            "token":"程",
            "start_offset":2,
            "end_offset":3,
            "type":"<IDEOGRAPHIC>",
            "position":2
        },
        {
            "token":"序",
            "start_offset":3,
            "end_offset":4,
            "type":"<IDEOGRAPHIC>",
            "position":3
        },
        {
            "token":"员",
            "start_offset":4,
            "end_offset":5,
            "type":"<IDEOGRAPHIC>",
            "position":4
        }
    ]
}
```

默认的中文分词是将每个字看成一个词，这显然是不符合要求的，所以我们需要安装中 文分词器来解决这个问题。 IK分词是一款国人开发的相对简单的中文分词器。虽然开发者自2012年之后就不在维护 了，但在工程应用中IK算是比较流行的一款！我们今天就介绍一下IK中文分词器的使用。

### **4.2 IK**分词器安装 

下载地址：https://github.com/medcl/elasticsearch-analysis-ik/releases 下载5.6.8版 本 课程配套资源也提供了: 资源\配套软件\elasticsearch\elasticsearch-analysis-ik- 5.6.8.zip

- 先将其解压，将解压后的elasticsearch文件夹重命名文件夹为ik 
- 将ik文件夹拷贝到elasticsearch/plugins 目录下。 
- 重新启动，即可加载IK分词器

### **4.3 IK**分词器测试

IK提供了两个分词算法ik_smart 和 ik_max_word 
其中 ik_smart 为最少切分，ik_max_word为最细粒度划分 
我们分别来试一下 
（1）最小切分：在浏览器地址栏输入地址 http://127.0.0.1:9200/_analyze?analyzer=ik_smart&pretty=true&text=我是程序员 
输出的结果为：

```json
{
    "tokens":[
        {
            "token":"我",
            "start_offset":0,
            "end_offset":1,
            "type":"CN_CHAR",
            "position":0
        },
        {
            "token":"是",
            "start_offset":1,
            "end_offset":2,
            "type":"CN_CHAR",
            "position":1
        },
        {
            "token":"程序员",
            "start_offset":2,
            "end_offset":5,
            "type":"CN_WORD",
            "position":2
        }
    ]
}
```

（2）最细切分：在浏览器地址栏输入地址

http://127.0.0.1:9200/_analyze?analyzer=ik_max_word&pretty=true&text=我是程序 
员输出的结果为：

```json
{
    "tokens":[
        {
            "token":"我",
            "start_offset":0,
            "end_offset":1,
            "type":"CN_CHAR",
            "position":0
        },
        {
            "token":"是",
            "start_offset":1,
            "end_offset":2,
            "type":"CN_CHAR",
            "position":1
        },
        {
            "token":"程序员",
            "start_offset":2,
            "end_offset":5,
            "type":"CN_WORD",
            "position":2
        },
        {
            "token":"程序",
            "start_offset":2,
            "end_offset":4,
            "type":"CN_WORD",
            "position":3
        },
        {
            "token":"员",
            "start_offset":4,
            "end_offset":5,
            "type":"CN_CHAR",
            "position":4
        }
    ]
}
```

### **4.4** 自定义词库 

我们现在测试"我是中国人"，浏览器的测试效果如下：

http://127.0.0.1:9200/_analyze?analyzer=ik_smart&pretty=true&text=我是中国人

```json
{
    "tokens":[
        {
            "token":"我",
            "start_offset":0,
            "end_offset":1,
            "type":"CN_CHAR",
            "position":0
        },
        {
            "token":"是",
            "start_offset":1,
            "end_offset":2,
            "type":"CN_CHAR",
            "position":1
        },
        {
            "token":"中",
            "start_offset":2,
            "end_offset":3,
            "type":"CN_CHAR",
            "position":2
        },
        {
            "token":"国",
            "start_offset":3,
            "end_offset":4,
            "type":"CN_CHAR",
            "position":3
        }
    ]
}
```

## **5 elasticsearch**与**MySQL**数据同步

### 5.1 **Logstash** 

#### **6.1.1** 什么是**Logstash** 

Logstash是一款轻量级的日志搜集处理框架，可以方便的把分散的、多样化的日志搜集 起来，并进行自定义的处理，然后传输到指定的位置，比如某个服务器或者文件。

#### **6.1.2 Logstash**安装与测试

解压，进入bin目录 

```shell
logstash ‐e 'input { stdin { } } output { stdout {} }'
```

stdin，表示输入流，指从键盘输入 

stdout，表示输出流，指从显示器输出

命令行参数: 

-e 执行

--config 或 -f 配置文件，后跟参数类型可以是一个字符串的配置或全路径文件名或全路径 路径(如：/etc/logstash.d/，logstash会自动读取/etc/logstash.d/目录下所有*.conf 的文 本文件，然后在自己内存里拼接成一个完整的大配置文件再去执行)

### **6.2 MySQL**数据导入**Elasticsearch** 

（1）在logstash-5.6.8安装目录下创建文件夹mysqletc （名称随意） 

（2）文件夹下创建mysql.conf （名称随意） ，内容如下：

```tex
input { 
	 jdbc { 
	 	# mysql jdbc connection string to our backup databse 后面的test对应mysql中的test数据库
	 	jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/tensquare_article?characterEncoding=UTF8"
	 	# the user we wish to excute our statement as
	 	 jdbc_user => "root" 
	 	 jdbc_password => "123456" 
	 	 # the path to our downloaded jdbc driver   
	 	 jdbc_driver_library => "D:/logstash‐5.6.8/mysqletc/mysql‐connector‐java‐5.1.46.jar"
	 	 # the name of the driver class for mysql
	 	 jdbc_driver_class => "com.mysql.jdbc.Driver"
	 	 jdbc_paging_enabled => "true"
	 	 jdbc_page_size => "50000"
	 	 #以下对应着要执行的sql的绝对路径。
	 	 statement => "select id,title,content from tb_article"
	 	 #定时字段 各字段含义（由左至右）分、时、天、月、年，全部为*默认含义为每分钟都更新
	 	 schedule => "* * * * *"
	 }
}


output {
	elasticsearch {
		#ESIP地址与端口
		hosts => "localhost:9200"
		#ES索引名称（自己定义的）
		index => "tensquare"
		#自增ID编号
		document_id => "%{id}"
		document_type => "article"
	}
	stdout {
		#以JSON格式输出
		codec => json_lines
	}
}
```

（3）将mysql驱动包mysql-connector-java-5.1.46.jar拷贝至D:/logstash- 5.6.8/mysqletc/ 下 。D:/logstash-5.6.8是你的安装目录

（4）命令行下执行 

```shell
logstash ‐f ../mysqletc/mysql.conf
```

观察控制台输出，每间隔1分钟就执行一次sql查询。