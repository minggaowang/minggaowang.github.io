---
layout:     post
title:      Spring 生命周期之 Bean 销毁
subtitle:   Bean 销毁
date:       2020-03-24
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 Spring Bean 销毁阶段划分

- Bean 销毁前阶段
- Bean 销毁阶段
- Bean 垃圾回收（GC）

## 2 Spring Bean 销毁各阶段详细分析

### 2.1 Bean 销毁前阶段

DestructionAwareBeanPostProcessor#postProcessBeforeDestruction

#### 2.1.1 代码示例

```java
...
public class MyDestructionAwareBeanPostProcessor implements DestructionAwareBeanPostProcessor{

    @Override
    public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {

        if (ObjectUtils.nullSafeEquals("userHolder", beanName) && UserHolder.class.equals(bean.getClass())) {{

            UserHolder userHolder = (UserHolder) bean;
            // afterSingletonsInstantiated() = the user holder v8
            userHolder.setDescription("the user holder v9");
            System.out.println("postProcessBeforeDestruction() = " + userHolder.getDescription());
        }}
    }
}    
```

#### 2.1.2 源码讲解

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
...
    @Override
	public void destroyBean(String beanName, Object beanInstance) {
		destroyBean(beanName, beanInstance, getMergedLocalBeanDefinition(beanName));
	}
...
    protected void destroyBean(String beanName, Object bean, RootBeanDefinition mbd) {
		new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), getAccessControlContext()).destroy();
	}
...    
}
```

```java
class DisposableBeanAdapter implements DisposableBean, Runnable, Serializable {
...
@Override
	public void destroy() {
		if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
			for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
				processor.postProcessBeforeDestruction(this.bean, this.beanName);
			}
		}
		...
	}    
...
}
```

### 2.2 Bean 销毁阶段

- @PreDestroy 标准方法
- 实现 DisposableBean 接口的 destroy() 方法
- 自定义销毁方法

#### 2.2.1 代码示例

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

    <bean class="org.wmg.thinking.in.spring.bean.lifecycle.MyInstantiationAwareBeanPostProcessor" />

    <bean id = "userHolder" class="org.wmg.thinking.in.spring.bean.lifecycle.UserHolder" autowire="constructor" destroy-method="doDestroy">
        <!-- 通过 XML 元素配置 -->
        <property name="description" value="the user holder" />
    </bean>
</beans>
```



```java
public class UserHolder implements DisposableBean {
    ...
    @PreDestroy
    public void preDestroy() {

        // postProcessBeforeDestruction() = the user holder v9
        this.description = "the user holder v10";
        System.out.println("preDestroy() = " + description);
    }

    @Override
    public void destroy() throws Exception {

        // preDestroy() = "the user holder v11";
        this.description = "the user holder v11";
        System.out.println("destroy() = " + description);
    }

    public void doDestroy(){

        // destroy = "the user holder v12";
        this.description = "the user holder v12";
        System.out.println("doDestroy() = " + description);
    }
    ...
}
```

#### 2.2.2 源码讲解

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
...
    @Override
	public void destroyBean(String beanName, Object beanInstance) {
		destroyBean(beanName, beanInstance, getMergedLocalBeanDefinition(beanName));
	}
...
    protected void destroyBean(String beanName, Object bean, RootBeanDefinition mbd) {
		new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), getAccessControlContext()).destroy();
	}
...    
}
```

```java
class DisposableBeanAdapter implements DisposableBean, Runnable, Serializable {
...
@Override
	public void destroy() {
		...
        if (this.destroyMethod != null) {
			invokeCustomDestroyMethod(this.destroyMethod);
		}
    	...
	}    
...
}
```

### 2.3 Bean 垃圾回收（GC）

- 关闭 Spring 容器（应用上下文）
- 执行 GC
- Spring Bean 覆盖 finalize() 方法被回调

#### 2.3.1 代码示例

```java
public class UserHolder implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware, EnvironmentAware,
        InitializingBean, SmartInitializingSingleton, DisposableBean {
 	...
    protected void finalize() throws Throwable {

        System.out.println("UserHolder is finalzied...");
    }
 }
```

```java
public class BeanLifecycleDemo {

    public static void main(String[] args) throws InterruptedException {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        beanFactory.addBeanPostProcessor(new MyInstantiationAwareBeanPostProcessor());
        // 添加 MyDestructionAwareBeanPostProcessor 执行销毁前回调
        beanFactory.addBeanPostProcessor(new MyDestructionAwareBeanPostProcessor());
        // 添加 CommonAnnotationBeanPostProcessor 解决 @PostConstructor
        beanFactory.addBeanPostProcessor(new CommonAnnotationBeanPostProcessor());
        // 基于 XML 资源 BeanDefinitionReader 实现
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        String[] locations = {"META-INF/dependency-lookup-context.xml",
                "META-INF/bean-constructor-dependency-injection.xml"};
        int beanNumbers = beanDefinitionReader.loadBeanDefinitions(locations);
        System.out.println("已加载 BeanDefinition 数量：" + beanNumbers);

        // 显示执行 preInstantiateSingletons()
        // SmartInitliziingSingleton 通常在 Spring ApplicationContext 场景使用。
        // preInstantiateSingletons() 将已注册的 BeanDefinition 初始化成 Spring Bean
        beanFactory.preInstantiateSingletons();

        // 通过 Bean id 和类型进行依赖查找
        User user = beanFactory.getBean("user", User.class);
        System.out.println(user);

        User superUser = beanFactory.getBean("superUser", User.class);
        System.out.println(superUser);

        // 构造器注入按照类型注入，resolveDependency
        UserHolder userHolder = beanFactory.getBean("userHolder", UserHolder.class);
        System.out.println(userHolder);

        //执行 Bean 销毁（容器内）
        beanFactory.destroyBean("userHolder", userHolder);
        // Bean  销毁并不意味着 Bean 垃圾回收了
        System.out.println(userHolder);

        // 销毁 BeanFactory 中的单例 Bean
        beanFactory.destroySingletons();;

        //强制 GC
        System.gc();

        // 等待一段时间
        Thread.sleep(1000L);

        //强制 GC
        System.gc();
    }
}
```

