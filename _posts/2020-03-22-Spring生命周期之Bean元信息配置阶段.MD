---
layout:     post
title:      Spring 生命周期之 Bean 元信息配置阶段
subtitle:   Bean 元信息配置阶段
date:       2020-03-22
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 Bean 元信息配置阶段作用

可以通过外部化配置方式读取，将 Spring Bean 注入到 IOC 容器，从而可以通过依赖查找方式获取到 Spring Bean 对象。

## 2 Bean 元信息配置几种方式

- PropertiesBeanDefinitionReader（ Properties 文件读取）
  - AbstractBeanDefinitionReader
    - BeanDefinitionReader
    - EnvironmentCapable
- XmlBeanDefinitionReader（ XML 文件读取）
  - AbstractBeanDefinitionReader
    - BeanDefinitionReader
    - EnvironmentCapable
- GroovyBeanDefinitionReader（ Groovy 文件读取）
  - AbstractBeanDefinitionReader
    - BeanDefinitionReader
    - EnvironmentCapable
  - GroovyObject
- AnnotatedBeanDefinitionReader（注解方式读取）

## 3 Bean 元信息配置读取示例

### 3.1 PropertiesBeanDefinitionReader（ Properties 文件读取）示例

```java
...
public class BeanMetadataConfigurationDemo {

    public static void main(String[] args) {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 实例化基于 Properties 资源 BeanDefinitionReader
        PropertiesBeanDefinitionReader propertiesBeanDefinitionReader = new PropertiesBeanDefinitionReader(beanFactory);
        String location = "META-INF/user.properties";
        // 基于 ClassPath 加载 Properties 资源
        Resource resource = new ClassPathResource(location);
        // 指定字符编码 UTF-8
        EncodedResource encodedResource = new EncodedResource(resource, "UTF-8");
        int beanNumbers = propertiesBeanDefinitionReader.loadBeanDefinitions(encodedResource);

        System.out.println("已加载 BeanDefinition 数量：" + beanNumbers);

        // 通过 Bean Id 和 类型进行依赖查找
        User user = beanFactory.getBean("user", User.class);
        System.out.println(user);
    }
}    
```

### 3.2 XmlBeanDefinitionReader（ XML 文件读取）示例

```java
...
public class MergedBeanDefinitionMergeDemo {

    public static void main(String[] args) {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 基于 XML 资源 BeanDefinition 实现
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        String location = "META-INF/dependency-lookup-context.xml";
        // 基于 ClassPath 加载 XML 资源
        Resource resource = new ClassPathResource(location);
        // 指定字符集编码 UTF-8
        EncodedResource encodedResource = new EncodedResource(resource, "UTF-8");
        int beanNumbers = beanDefinitionReader.loadBeanDefinitions(encodedResource);
        System.out.println("已加载 BeanDefinition 数量：" + beanNumbers);
        // 通过 Bean Id 和类型进行依赖查找
        User user = beanFactory.getBean("user", User.class);
        System.out.println(user);

        User superUser = beanFactory.getBean("superUser", User.class);
        System.out.println(superUser);
    }
}    
```

### 3.3 AnnotatedBeanDefinitionReader（注解方式读取）示例

```java
...
public class AnnotatedBeanDefinitionParsingDemo {

    public static void main(String[] args) {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 基于 Java 注解的 BeanDefinitionReader 的实现
        AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(beanFactory);
        int beanDefinitionCountBefore = beanFactory.getBeanDefinitionCount();
        // 注册当前类（非 @Component class）
        annotatedBeanDefinitionReader.register(AnnotatedBeanDefinitionParsingDemo.class);
        int beanDefinitionCountAfter = beanFactory.getBeanDefinitionCount();
        int beanDefinitionCount = beanDefinitionCountAfter - beanDefinitionCountBefore;
        System.out.println("已加载 BeanDefinition 数量：" + beanDefinitionCount);
        // 普通的 Class 作为 Component 注册到 Spring IOC 容器后，通常 Bean 名称为 annotatedBeanDefinitionParsingDemo
        // Bean 名称生成来自于 BeanNameGenerator，注解实现 AnnotationBeanNameGenerator
        AnnotatedBeanDefinitionParsingDemo demo = beanFactory.getBean("annotatedBeanDefinitionParsingDemo",
                AnnotatedBeanDefinitionParsingDemo.class);
        System.out.println(demo);
    }
}    
```

## 4 总结

Bean 信息被读取之后，为后续 Bean 注册提供元信息。