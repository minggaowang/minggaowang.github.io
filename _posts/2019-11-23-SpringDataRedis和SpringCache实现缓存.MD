---
layout:     post
title:      SpringDataRedis和SpringCache实现缓存
subtitle:   spring缓存
date:       2019-11-23
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - SpringData
    - Spring
---

## 使用场景

当访问比较高时，直接连接数据库会造成并发数过多，最后导致数据库宕掉，这需要用到缓存机制。SpringDataRedis 和 SpringCache 都能实现缓存。前者比较强大，能够支持复杂的数据操作，并且可以缓存到其他环境的JVM内存；后者只能支持当前环境下的JVM内存。

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring-data-3.jpg)

## SpringDataRedis实现缓存（一）

pom.xml引入依赖

```xml
...
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
...
```

application.yml 中加入

```yml
spring: 
  application:  
    name: demo-article #指定服务名
  datasource:  
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://192.168.1.2:3306/demo_article?characterEncoding=UTF8
    username: root
    password: 123456
  jpa: 
    database: MySQL
    show-sql: true
  redis:
    host: 192.168.1.2
```

```java
...
@Service
@Transactional
public class ArticleService {
    
    @Autowired
	private ArticleDao articleDao;
	
	@Autowired
	private IdWorker idWorker;

	@Autowired
	private RedisTemplate redisTemplate;
    
    ....

	/**
	 * 根据ID查询实体
	 * @param id
	 * @return
	 */
	public Article findById(String id) {
		//先从缓存中拿数据
		Article article = (Article)redisTemplate.opsForValue().get("article_"+id);
		//如果拿不到，就去数据库中查询
		if (article==null){
			article = articleDao.findById(id).get();
			//放入缓存中
			redisTemplate.opsForValue().set("article_"+id, article, 20, TimeUnit.SECONDS);
		}
		return article;
	}
        
	/**
	 * 修改
	 * @param article
	 */
	public void update(Article article) {
		//清除缓存中的数据
		redisTemplate.delete("article_"+article.getId());
		articleDao.save(article);
	}
    
        
    /**
	 * 删除
	 * @param id
	 */
	public void deleteById(String id) {
		//清除缓存中的数据
		redisTemplate.delete("article_"+id);
		articleDao.deleteById(id);
	}
    
    ...
}
```

## SpringCache实现缓存（二）

Spring Cache使用方法与Spring对事务管理的配置相似。Spring Cache的核心就是对某 个方法进行缓存，其实质就是缓存该方法的返回结果，并把方法参数和结果用键值对的 方式存放到缓存中，当再次调用该方法使用相应的参数时，就会直接从缓存里面取出指 定的结果进行返回。

常用注解：

- @Cacheable-------使用这个注解的方法在执行后会缓存其返回结果。
- @CacheEvict--------使用这个注解的方法在其执行前或执行后移除Spring Cache中的某些 元素。

```java
...
@Service
public class GatheringService {
    
    @Autowired
	private GatheringDao gatheringDao;
	
	@Autowired
	private IdWorker idWorker;
    
    ...
        
    /**
	 * 根据ID查询实体
	 * @param id
	 * @return
	 * @Cacheable 添加缓存 value属性表示缓存整体唯一标识,key属性标识缓存键值对中的key
	 */
	@Cacheable(value = "gathering",key = "#id")
	public Gathering findById(String id) {
		return gatheringDao.findById(id).get();
	}
    
    /**
	 * 修改
	 * @param gathering
	 */
	@CacheEvict(value = "gathering", key = "#gathering.id")
	public void update(Gathering gathering) {
		gatheringDao.save(gathering);
	}
    
    /**
	 * 删除
	 * @param id
	 */
	@CacheEvict(value = "gathering", key = "#gathering.id")
	public void deleteById(String id) {
		gatheringDao.deleteById(id);
	}
    
    ...
}
```