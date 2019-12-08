---
layout:     post
title:      SpringIOC注解使用
subtitle:   SpringIOC
date:       2019-12-08
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 xml配置文件的形式VS配置类的形式

### 1.1 基于 xml 的形式定义 Bean 的信息

```xml
<?xml version="1.0" encoding="utf-8"?>

<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"> /定义一个Bean的信息 
  <bean id="car" class="com.demo.compent.Car"/> 
</beans>
```

> 去容器中读取Bean

```java
...
public static void main( String[] args ){
    ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
    System.out.println(ctx.getBean("person"));
}
```

### 1.2 基于读取配置类的形式定义Bean信息

```java
...
@Configuration
public class MainConfig {
    
    @Bean
    public Person person(){
        return new Person();
    }
}    
```

>注意: 通过@Bean的形式是使用的话， bean的默认名称是方法名，若@Bean(value="bean的名称") 那么bean的名称是指定的

> 去容器中读取Bean的信息（传入配置类）

```java
...
public static void main( String[] args ){
    
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
    System.out.println(ctx.getBean("person"));
}
```

## 2 在配置类上写@CompentScan注解来进行包扫描

```java
...
@Configuration
@ComponentScan(basePackages = {"com.tuling.testcompentscan"})
public class MainConfig {
    ...
}    
```

### 2.1 排除用法excludeFilters(排除@Controller注解的,和DemoService的)

```java
...
@Configuration
@ComponentScan(basePackages = {"com.tuling.testcompentscan"},excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class}),@ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,value = {TulingService.class})})
public class MainConfig {
    ...
}    
```

### 2.2 :包含用法 includeFilters ,注意，若使用包含的用法，需要把useDefaultFilters属性设置为false（true表示扫描全部的）

```java
@Configuration
@ComponentScan(basePackages = {"com.tuling.testcompentscan"},includeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class, Service.class})},useDefaultFilters = false)
public class MainConfig {
    ...
}
```

### 2.3 @ComponentScan.Filter type的类型

- 注解形式的FilterType.ANNOTATION @Controller @Service @Repository @Compent

- 指定类型的 FilterType.ASSIGNABLE_TYPE @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,value = {TulingService.class})

- aspectj类型的 FilterType.ASPECTJ(不常用)

- 正则表达式的 FilterType.REGEX(不常用)

- 自定义的 FilterType.CUSTOM

  ```java
  public enum FilterType {
      //注解形式 比如@Controller @Service @Repository @Compent
      ANNOTATION,
      //指定的类型
      ASSIGNABLE_TYPE,
      //aspectJ形式的
      ASPECTJ,
      //正则表达式的
      REGEX,
      //自定义的
      CUSTOM
  }
  ```

  FilterType.CUSTOM 自定义类型如何使用

  ```java
  public class DemoFilterType implements TypeFilter {
      
      @Override
      public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException{
          //获取当前类的注解源信息
          AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
          
          //获取当前类的class的源信息
          ClassMetadata classMetadata = metadataReader.getClassMetadata();
          //获取当前类的资源信息
          Resource resource = metadataReader.getResource();
          if(classMetadata.getClassName().contains("dao")) {
              return true;
          }
          return false;
      }
  }
  ```

  ```java
  ...
  @ComponentScan(basePackages = {"com.demo.testcompentscan"},includeFilters = {
      @ComponentScan.Filter(type = FilterType.CUSTOM,value = DemoFilterType.class)
  },useDefaultFilters = false)
  public class MainConfig {
      ...
  }    
  ```

## 3 配置Bean的作用域对象

### 3.1 在不指定@Scope的情况下，所有的bean都是单实例的bean,而且是饿汉加载(容器启动实例就创建 好了)

```java
@Bean
public Person person() {
    return new Person();
}
```

### 3.2 指定@Scope为 prototype 表示为多实例的，而且还是懒汉模式加载（IOC容器启动的时候，并不会创建对象，而是 在第一次使用的时候才会创建）

```java
@Bean
@Scope(value = "prototype")
public Person person() {
    return new Person();
}
```

> singleton 单实例的(默认) 
>
> prototype 多实例的
>
> request 同一次请求
>
> session 同一个会话级别

### 3.3 Bean的懒加载@Lazy(主要针对单实例的bean 容器启动的时候，不创建对象，在第一次使用的时候才会创建该对象)

```java
@Bean
@Lazy
public Person person() {
    return new Person();
}
```

## 4 @Conditional进行条件判断

> 场景,有二个组件DemoAspect 和DemoLog ，我的DemoLog组件是依赖于DemoAspect的组件

应用:自己创建一个DemoCondition的类 实现Condition接口

```java
...
public class TulingCondition implements Condition {
    /**
     *
     * @param context
     * @param metadata
     * @return
     */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        //判断容器中是否有demoAspect的组件
        if(context.getBeanFactory().containsBean("demoAspect")) {
            return true;
        }
        return false;
    }
}
```

```java
...
public class MainConfig {
    
    @Bean
    public DemoAspect demoAspect() {
        return new DemoAspect();
    }
    
    //当切 容器中有demoAspect的组件，那么demoLog才会被实例化.
    @Bean
    @Conditional(value = DemoCondition.class)
    public DemoLog demoLog() {
        return new DemoLog();
    }
}
```

## 5 往IOC 容器中添加组件的方式

### 5.1 通过@CompentScan +@Controller @Service @Respository @compent

> 适用场景: 针对我们自己写的组件可以通过该方式来进行加载到容器中。

### 5.2 通过@Bean的方式来导入组件(适用于导入第三方组件的类)

### 5.3 通过@Import来导入组件 （导入组件的id为全类名路径）

```java
...
@Configuration
@Import(value = {Person.class, Car.class})
public class MainConfig {
    ...
}
```

> 通过@Import 的ImportSeletor类实现组件的导入 (导入组件的id为全类名路径)

```java
...
public classDemoImportSelector implements ImportSelector {
    //可以获取导入类的注解信息
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.demo.testimport.compent.Dog"};
    }
}    
```

```java
...
@Configuration
@Import(value = {Person.class, Car.class, DemoImportSelector.class})
public class MainConfig {
    ...
}    
```

> 通过@Import的 ImportBeanDefinitionRegister导入组件 (可以指定bean的名称)

```java
...
public class DemoBeanDefinitionRegister implements Import
    BeanDefinitionRegistrar {
    
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //创建一个bean定义对象
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Cat.class);
        //把bean定义对象导入到容器中
        registry.registerBeanDefinition("cat",rootBeanDefinition); }
    }
}    
```

```java
...
@Configuration
@Import(value = {Person.class, Car.class, DemoImportSelector.class, DemoBeanDefinitionRegister.class})
public class MainConfig {
    ...
}    
```

### 5.4 通过实现FacotryBean接口来实现注册组件

```java
...
public class CarFactoryBean implements FactoryBean<Car> {
    //返回bean的对象
    @Override
    public Car getObject() throws Exception {
        return new Car();
    }
    //返回bean的类型
    @Override
    public Class<?> getObjectType() {
        return Car.class;
    }
    //是否为单利
    @Override
    public boolean isSingleton() {
        return true;
    }
}    
```

## 6 Bean的初始化方法和销毁方法

### 6.1 什么是bean的生命周期

>bean的创建----->初始化----->销毁方法
>
>由容器管理Bean的生命周期，我们可以通过自己指定bean的初始化方法和bean的销毁方法
>
>```java
>...
>@Configuration
>public class MainConfig {
>    //指定了bean的生命周期的初始化方法和销毁方法.
>    @Bean(initMethod = "init",destroyMethod = "destroy")
>    public Car car() {
>        return new Car();
>    }
>}    
>```
>
>针对单实例bean的话，容器启动的时候，bean的对象就创建了，而且容器销毁的时候，也会调用Bean的销毁方法
>
>针对多实例bean的话,容器启动的时候，bean是不会被创建的而是在获取bean的时候被创建，而且bean的销毁不受 IOC容器的管理

### 6.2 通过 InitializingBean和DisposableBean 的二个接口实现bean的初始化以及销毁方法

```java
...
@Component
public class Person implements InitializingBean,DisposableBean {
    public Person() {
        System.out.println("Person的构造方法");
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean的destroy()方法 ");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean的 afterPropertiesSet方法");
    }
}    
```

### 6.3 通过JSR250规范 提供的注解@PostConstruct 和@ProDestory标注的方法

```java
...
@Component
public class Book {
    public Book() {
        System.out.println("book 的构造方法");
    }
    @PostConstruct
    public void init() {
        System.out.println("book 的PostConstruct标志的方法");
    }
    @PreDestroy
    public void destory() {
        System.out.println("book 的PreDestory标注的方法");
    }
}    
```

### 6.4 通过Spring的BeanPostProcessor的 bean的后置处理器会拦截所有bean创建过程

> postProcessBeforeInitialization 在init方法之前调用
>
> postProcessAfterInitialization 在init方法之后调用

```java
...
@Component
public class DemoBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("DemoBeanPostProcessor...postProcessBeforeInitialization:"+beanName);
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("DemoBeanPostProcessor...postProcessAfterInitialization:"+beanName);
        return bean;
    }
}    
```

## 7 通过@Value +@PropertySource来给组件赋值

```java
...
public class Person {
    //通过普通的方式
    @Value("小明")
    private String firstName;
    
    //spel方式来赋值
    @Value("#{28-8}")
    private Integer age;
    
    //通过读取外部配置文件的值
    @Value("${person.lastName}")
    private String lastName;
    ...
}    
```

```java
...
@Configuration
@PropertySource(value = {"classpath:person.properties"}) //指定外部文件的位置
public class MainConfig {
    
    @Bean
    public Person person() {
        return new Person();
    }
}    
```

## 8 自动装配

### 8.1 @AutoWired的使用

自动注入

```java
...
//一个Dao
@Repository
public class DemoDao {
    ...
}    
```

```java
...
//一个Service
@Service
public class DemoService {
    
    @Autowired
    private TulingDao tulingDao;
    ...
}    
```

> 结论:
>
> 自动装配首先时按照类型进行装配，若在IOC容器中发现了多个相同类型的组件，那么就按照 属性名称来进行装配
>
> @Autowired
>
> private DemoDao demoDao;
>
> 比如，我容器中有二个DemoDao类型的组件 一个叫demoDao 一个叫demoDao2
>
> 那么我们通过@AutoWired 来修饰的属性名称时tulingDao，那么拿就加载容器的tulingDao组件，若属性名称为 tulignDao2 那么他就加载的时tulingDao2组件
>
> 假设我们需要指定特定的组件来进行装配，我们可以通过使用@Qualifier("tulingDao")来指定装配的组件 或者在配置类上的@Bean加上@Primary注解
>
> ```java
> ...
> @Autowired
> @Qualifier("tulingDao")
> private TulingDao tulingDao2;
> ...
> ```
>
> 假设我们容器中即没有demoDao 和demoDao2,那么在装配的时候就会抛出异常
>
> No qualifying bean of type 'com.demo.testautowired.TulingDao' available
>
> 若我们想不抛异常 ，我们需要指定 required为false的时候可以了
>
> ```java
> ...
> @Autowired(required = false)
> @Qualifier("demoDao")
> private DemoDao demoDao2;
> ...
> ```
>
> 标注在set方法上
>
> ```java
> @Autowired
> public void setDemoLog(DemoLog demoLog) {
>     this.demoLog = demoLog;
> }
> ```
>
> 标注在构造方法上 
>
> ```java
> ...
> @Autowired
> public DemoAspect(DemoLog demoLog) {
>     this.demoLog = demoLog;
> }    
> ```
>
> 标注在配置类上的入参中（可以不写）
>
> ```java
> @Bean
> public DemoAspect demoAspect(@Autowired DemoLog demoLog) {
> 	DemoAspect demoAspect = new DemoAspect(demoLog);
>     return demoAspect;
> }
> ```
>
> 

## 8.2 @Resource(JSR250规范)

> 功能和@AutoWired的功能差不多一样，但是不支持@Primary 和@Qualifier的支持

## 8.3 @InJect（JSR330规范）

> 需要导入jar包依赖

> 功能和支持@Primary功能 ,但是没有Require=false的功能

```xml
<?xml version="1.0" encoding="utf-8"?>

<dependency> 
  <groupId>javax.inject</groupId>  
  <artifactId>javax.inject</artifactId>  
  <version>1</version> 
</dependency>
```

## 9 我们自己的组件 需要使用spring ioc的底层组件的时候,比如 ApplicationContext等

### 9.1 我们可以通过实现XXXAware接口来实现

```java
...
@Component
public class DemoCompent implements ApplicationContextAware,BeanNameAware {
    
    private ApplicationContext applicationContext;
    
    @Override
    public void setBeanName(String name) {
        System.out.println("current bean name is :【"+name+"】");
    }
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}    
```

### 9.2 **通过**@Profile**注解 来根据环境来激活标识不同的**Bean

> @Profile标识在类上，那么只有当前环境匹配，整个配置类才会生效

> @Profile标识在Bean上 ，那么只有当前环境的Bean才会被激活

> 没有标志为@Profile的bean 不管在什么环境都可以被激活

```java
...
@Configuration
@PropertySource(value = {"classpath:ds.properties"})
public class MainConfig implements EmbeddedValueResolverAware {
    
    @Value("${ds.username}")
    private String userName;
    
    @Value("${ds.password}")
    private String password;
    
    private String jdbcUrl;
    
    private String classDriver;
    
    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.jdbcUrl = resolver.resolveStringValue("${ds.jdbcUrl}");
        this.classDriver = resolver.resolveStringValue("${ds.classDriver}");
    }
    
    //标识为测试环境才会被装配
    @Bean
    @Profile(value = "test")
    public DataSource testDs() {
        return buliderDataSource(new DruidDataSource());
    }
    
    //标识开发环境才会被激活
    @Bean
    @Profile(value = "dev")
    public DataSource devDs() {
        return buliderDataSource(new DruidDataSource());
    }
    //标识生产环境才会被激活
    @Bean
    @Profile(value = "prod")
    public DataSource prodDs() {
        return buliderDataSource(new DruidDataSource());
    }
    
    private DataSource buliderDataSource(DruidDataSource dataSource) {
        dataSource.setUsername(userName);
        dataSource.setPassword(password);
        dataSource.setDriverClassName(classDriver);
        dataSource.setUrl(jdbcUrl);
        return dataSource;
    }
}    
```

> **激活切换环境的方法** 
>
> 方法一:通过运行时jvm参数来切换 -Dspring.profiles.active=test|dev|prod
>
> 方法二:通过代码的方式来激活
>
> ```java
> ...
> public static void main(String[] args) {
>     AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
>     ctx.getEnvironment().setActiveProfiles("test","dev");
>     ctx.register(MainConfig.class);
>     ctx.refresh();
>     printBeanName(ctx);
> }    
> 
> private static void printBeanName(AnnotationConfigApplicationContext ctx){
>         for(String beanName:ctx.getBeanDefinitionNames()) {
>             System.out.println("容器中的BeanName："+beanName);
>         }
>     }
> ...
> ```
>
> 