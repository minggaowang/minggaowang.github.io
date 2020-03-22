---
layout:     post
title:      Spring 生命周期之 Bean 合并阶段
subtitle:   Bean 合并阶段
date:       2020-03-22
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 BeanDefinition 合并

关于 Spring Bean 在初始化或实例化之前，有一个新的阶段就叫 BeanDefinition 合并阶段。

- 父子 BeanDefinitin 合并
  - 当前 BeanFactory 查找
  - 层次性 BeanFactory 查找

## 2 BeanDefinition 合并引用示例

### 2.1 存在父子关系的 Bean XML 定义

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans
        xmlns="http://www.springframework.org/schema/beans"
        xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/beans/spring-context.xsd">

<!--    <context:annotation-config />-->

<!--    <context:component-scan base-package="org.wmg.thinking.in.spring.ioc.overview.dependency.injection" />-->

    <!-- Root BeanDefinition 不需要合并，不存在 parent -->
    <!-- 普通 BeanDefinition GenericBeanDefinition -->
    <!-- 经过合并后 GenericBeanDefinition 变成 RootBeanDefinition-->
    <bean id="user" class="org.wmg.thinking.in.spring.ioc.overview.domain.User">
        <property name="id" value="1" />
        <property name="name" value="年糕" />
        <property name="city" value="HANGZHOU" />
        <property name="workCities" value="BEIJING,HANGZHOU" />
        <property name="lifeCities">
            <list>
                <value>BEIJING</value>
                <value>SHANGHAI</value>
            </list>
        </property>
        <property name="configFileLocation" value="classpath:/META-INF/user-config.properties" />
    </bean>

    <!-- 普通 beanDefinition GenericBeanDefinition -->
    <!-- 合并后 GenericBeanDefinition 变成 RootBeanDefinition，并且覆盖 parent 相关配置-->
    <!-- primary = true，增加了 address 属性-->
    <bean id="superUser" class="org.wmg.thinking.in.spring.ioc.overview.domain.SuperUser" parent="user" primary="true">
        <property name="address" value="上海" />
    </bean>

    <bean id="objectFactory" class="org.springframework.beans.factory.config.ObjectFactoryCreatingFactoryBean">
        <property name="targetBeanName" value="user" />
    </bean>
</beans>
```

### 2.2 XML 读取加载之后，会进行 BeanDefinition 合并

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
```

## 3 BeanDefinition 合并（源码分析）

合并入口方法：AbstractBeanFactory#getMergedBeanDefinition(java.lang.String)

```java
...
/**
	 * Return a 'merged' BeanDefinition for the given bean name,
	 * merging a child bean definition with its parent if necessary.
	 * <p>This {@code getMergedBeanDefinition} considers bean definition
	 * in ancestors as well.
	 * @param name the name of the bean to retrieve the merged definition for
	 * (may be an alias)
	 * @return a (potentially merged) RootBeanDefinition for the given bean
	 * @throws NoSuchBeanDefinitionException if there is no bean with the given name
	 * @throws BeanDefinitionStoreException in case of an invalid bean definition
	 */
@Override
public BeanDefinition getMergedBeanDefinition(String name) throws BeansException {
		String beanName = transformedBeanName(name);
		// Efficiently check whether bean definition exists in this factory.
		if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {
			return ((ConfigurableBeanFactory) getParentBeanFactory()).getMergedBeanDefinition(beanName);
		}
		// Resolve merged bean definition locally.
		return getMergedLocalBeanDefinition(beanName);
}
...
```

## 4 总结

如果是根 BeanDefinition 即 RootBeanDefinition 不需要合并，不存在 parent。如果是普通 BeanDefinition 即 GenericBeanDefinition 需要合并，存在 parent。经过合并后 GenericBeanDefinition 变成 RootBeanDefinition。这样合并的好处是：子类不需要复写父类的属性，减少了代码的冗余。

