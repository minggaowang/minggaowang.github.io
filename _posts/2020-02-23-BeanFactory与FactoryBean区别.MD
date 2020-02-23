---
layout:     post
title:      BeanFactory与FactoryBean区别
subtitle:   SpringIOC
date:       2019-12-14
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 什么是BeanFactory?

BeanFactory 是 Ioc 的底层容器，更具体的说应该是 DefaultListableBeanFactory ，该类实现 AutowireCapableBeanFactory 、ListableBeanFactory 和 BeanDefinitionRegistry 接口，从而实现了Ioc的依赖注入和依赖查找。

### 1.1 AutowireCapableBeanFactory

该接口是 BeanFactory 扩展，具有自动装配能力，并且能把这种能力暴露给外部应用，则需要实现此接口。

**简单应用**

```java
/**
 * Bean 实例化
 */
public class SpecialBeanInstantinationDemo {

    public static void main(String[] args) {

        //配置 XML 配置文件
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:/META-INF/special-bean-instantiation-context.xml");
        AutowireCapableBeanFactory beanFactory = applicationContext.getAutowireCapableBeanFactory();
        ServiceLoader serviceLoader = beanFactory.getBean("userFactoryServiceLoader", ServiceLoader.class);

        //通过 ServiceLoaderFactoryBean 方式
        displayServiceLoader(serviceLoader);

        //原生 serviceLoader 方式
        demoServiceLoader();

        //创建 UserFactory 对象，通过 AutowireCapableBeanFactory 方式
        UserFactory userFactory = beanFactory.createBean(DefaultUserFactory.class);
        System.out.println(userFactory.createUser());
    }

    public static void demoServiceLoader() {

        ServiceLoader<UserFactory> serviceLoader = ServiceLoader.load(UserFactory.class, Thread.currentThread().getContextClassLoader());
        displayServiceLoader(serviceLoader);
    }

    private static void displayServiceLoader(ServiceLoader<UserFactory> serviceLoader){

        Iterator<UserFactory> iterator = serviceLoader.iterator();
        while (iterator.hasNext()){
            UserFactory userFactory = iterator.next();
            System.out.println(userFactory.createUser());
        }
    }
}
```

### 1.2 ListableBeanFactory

该接口是 BeanFactory 扩展，可以枚举它们的所有bean实例，而不是根据需要逐个尝试bean查找，预加载所有 bean 定义(如基于 xml 的工厂) 的 BeanFactory，则需要实现此接口。

**简单示例：**

```java
/*
 * 依赖查找
 */
public class DependecyLookupDemo {

    public static void main(String[] args) {

        BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:/META-INF/dependecy-lookup-context.xml");
        lookupByType(beanFactory);
        lookupByColletionType(beanFactory);
        lookupByAnnotaionType(beanFactory);
//        lookupInRealTime(beanFactory);
//        lookupInLazy(beanFactory);
    }

    private static void lookupByAnnotaionType(BeanFactory beanFactory) {

        if (beanFactory instanceof ListableBeanFactory) {
            ListableBeanFactory listableBeanFactory = (ListableBeanFactory) beanFactory;
            Map<String, User> users = (Map) listableBeanFactory.getBeansWithAnnotation(Super.class);
            System.out.println("查找标注 @Super 所有的USser 集合对象：" + users);
        }
    }

    private static void lookupByColletionType(BeanFactory beanFactory) {

        if (beanFactory instanceof ListableBeanFactory) {
            ListableBeanFactory listableBeanFactory = (ListableBeanFactory) beanFactory;
            Map<String, User> users = listableBeanFactory.getBeansOfType(User.class);
            System.out.println("查找到的所有的USser 集合对象：" + users);
        }
    }

    private static void lookupByType(BeanFactory beanFactory) {

        User user = beanFactory.getBean(User.class);
        System.out.println("实时查找" + user);
    }

    private static void lookupInLazy(BeanFactory beanFactory) {

        ObjectFactory<User> objectFactory = (ObjectFactory<User>) beanFactory.getBean("objectFactory");
        User user = objectFactory.getObject();
        System.out.println("延迟查找" + user);
    }

    private static void lookupInRealTime(BeanFactory beanFactory) {

        User user = (User) beanFactory.getBean("user");
        System.out.println("实时查找" + user);
    }
}
```

### 1.3 BeanDefinitionRegistry

顾名思义就是对 BeanDefinition 进行注册。

**简单示例**

```java
//@Import方式
@Import(AnnotaionBeanDefinitionDemo.Config.class)
public class AnnotaionBeanDefinitionDemo {

    public static void main(String[] args) {

        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        //注册配置类
        applicationContext.register(AnnotaionBeanDefinitionDemo.class);

        //通过 BeanDefinition 注册API实现
        //命名 bean
        registerUserBeanDefinition(applicationContext, "niangao-user");
        //非命名
        registerUserBeanDefinition(applicationContext);
        //启动 Spring 应用上下文
        applicationContext.refresh();
        //依赖查找
        System.out.println("Config 类型的所有 Beans：" + applicationContext.getBeansOfType(Config.class));
        System.out.println("User 类型的所有 Beans：：" + applicationContext.getBeansOfType(User.class));
        applicationContext.close();
    }

    /**
     * 命名 Bean 的注册方式
     * @param registry
     * @param beanName
     */
    public static void registerUserBeanDefinition(BeanDefinitionRegistry registry, String beanName) {

        BeanDefinitionBuilder beanDefinitionBuilder = genericBeanDefinition(User.class);
        beanDefinitionBuilder
                .addPropertyValue("id", 1L)
                .addPropertyValue("name", "王明高");
        if (StringUtils.hasText(beanName)) {
            //命名注册
            registry.registerBeanDefinition(beanName, beanDefinitionBuilder.getBeanDefinition());
        } else {
            //非命名注册方式
            BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinitionBuilder.getBeanDefinition(), registry);
        }
    }

    public static void registerUserBeanDefinition(BeanDefinitionRegistry registry) {

        registerUserBeanDefinition(registry, null);
    }

    //@Component方式
    @Component
    public static class Config {

        //@Bean方式
        @Bean(name = {"user", "wangminggao-user"})
        public User user() {

            User user = new User();
            user.setId(1L);
            user.setName("王明高");
            return user;
        }
    }
```

## 2 什么是FactoryBean？

是创建 Bean 的一种方式，帮助实现复杂的初始化逻辑。

**简单示例**

```java
public class UserFactoryBean implements FactoryBean {

    @Override
    public Object getObject() throws Exception {

        User user = new User();
        user.setId(1L);
        user.setName("niangao");
        return user;
    }

    @Override
    public Class<?> getObjectType() {
        return User.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
```