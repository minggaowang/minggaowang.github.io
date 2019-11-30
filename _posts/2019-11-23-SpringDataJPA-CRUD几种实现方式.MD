---
layout:     post
title:      SpringDataJPA-CRUD几种实现方式
subtitle:   JPA
date:       2019-11-23
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - SpringData
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring-data-2.jpg)

## 准备工作

pom.xml引入依赖

```xml
...
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
...
```

application.yml 中加入

```yml
...
spring: 
  application:  
    name: demo-article #指定服务名
  datasource:  
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://192.168.1.2:3306/demo-article?characterEncoding=UTF8
    username: root
    password: 123456
  jpa: 
    database: MySQL
    show-sql: true
...    
```

## 使用Specification<T>接口实现（一）

```java
...
@Service
@Transactional
public class ArticleService {
    @Autowired
    private ArticleDao articleDao;

    @Autowired
    private IdWorker idWorker;
    
	...
    
	public Page<Label> pageQuery(Article article, int page, int size){
        //封装了一个分页对象，在springdatajpa中想要实现分页，直接传一个分页对象即可
        Pageable pageable = PageRequest.of(page-1, size);
        return articleDao.findAll(new Specification<Article>(){
            @Override
            public Predicate toPredicate(Root<Article> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
                List<Predicate> list = new ArrayList<>();
                if(hasText(article.getName())){
                    Predicate predicate = cb.like(root.get("name").as(String.class), "%" + article.getName() + "%");
                    list.add(predicate);
                }
                if(hasText(article.getState())){
                    Predicate predicate = cb.equal(root.get("state").as(String.class), article.getState());
                    list.add(predicate);
                }
                Predicate[] parr = new Predicate[list.size()];
                //把集合中的属性复制到数组中
                parr = list.toArray(parr);
                return cb.and(parr);
            }
        }, pageable);
    }
    
	...  
}
```

## 通过Dao方法名方式实现（二）

```java
...
public interface ArticleDao extends JpaRepository<Article,String>,JpaSpecificationExecutor<Article>{

	...
    
	List<Article> findTop6ByStateOrderByCreatetimeDesc(String state);
    
	List<Recruit> findTop6ByStateNotOrderByCreatetimeDesc(String state);   
    
	...
}
```

## 通过原生sql方式实现（三）

```java
...
public interface ArticleDao extends JpaRepository<Article,String>,JpaSpecificationExecutor<Article>{

	...
    
    @Query(value = "SELECT * FROM tb_article WHERE id IN (SELECT articleid FROM tb_pa WHERE articleid=?) ORDER BY replytime DESC", nativeQuery = true)
    public Page<Problem> selectArticlesPage(String articleid, Pageable pageable);
    
	@Modifying
    @Query(value = "UPDATE tb_article SET state = 1 WHERE id = ?", nativeQuery = true)
    public void examine(String id);
    
	...    
}
```

## 个人感悟

以上三中均可以对数据库进行操作，这表明JPA实现方式还是比较全面。但在实际的开发过程中，个人认为还是通过原生sql的方式比较好，sql语句在navicat premium 或者 SQLyog 运行通过后拷贝过来，修改参数值为？。