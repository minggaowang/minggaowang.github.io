---
layout:     post
title:      Spring 生命周期之 Bean 初始化阶段
subtitle:   Bean 初始化阶段
date:       2020-03-24
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 Bean 初始化阶段划分

- Bean 初始化前阶段
- Bean 初始化阶段
- Bean 初始化后阶段
- Bean 初始化完成阶段

## 2 Bean 初始化各阶段详细分析

### 2.1 Bean 初始化前阶段

BeanPostProcessor#postProcessBeforeInitialization

#### 2.1.1 代码示例

```java
...
class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    ...
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        if (ObjectUtils.nullSafeEquals("userHolder", beanName) && UserHolder.class.equals(bean.getClass())) {

            UserHolder userHolder = (UserHolder) bean;
            userHolder.setDescription("the user holder v3");
        }
        return bean;
    }    
   ...     
}    
```

#### 2.1.2 源码讲解

方法入口：AbstractAutowireCapableBeanFactory#initializeBean

```java
...
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    ...   
    if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}
    ...   
}  

...
    
@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
...
```

### 2.2 初始化阶段

- @PostConstruct 标注方法
- 实现 InitializingBean 接口的 afterPropertiesSet() 方法
- 自定义初始化方法

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

    <bean id = "userHolder" class="org.wmg.thinking.in.spring.bean.lifecycle.UserHolder" autowire="constructor" init-method="init">
        <!-- 通过 XML 元素配置 -->
        <property name="description" value="the user holder" />
    </bean>
</beans>
```



```java
...
public class UserHolder implements InitializingBean {
...
    @PostConstruct
    public void initPostConstruct() {

        // postProcessBeforeInitialization v3 -> initPostConstruct v4
        this.description = "the user holder v4";
        System.out.println("initPostConstruct() = " + description);
    }

    @Override
    public void afterPropertiesSet() throws Exception {

        // initPostConstruct v4 -> afterPropertiesSet v5
        this.description = "the user holder v5";
        System.out.println("afterPropertiesSet() = " + description);
    }

    /**
     * 自定义初始化方法
     */
    public void init() {

        // afterPropertiesSet v5 -> init v6
        this.description = "the user holder v6";
        System.out.println("init() = " + description);
    }
... 
}    
```

#### 2.2.2 源码讲解

```java
...
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    ...
    try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}    
    ...    
}
...
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}  
...
```

### 2.3 初始化后阶段

BeanPostProcessor#postProcessAfterInitialization

#### 2.3.1 代码示例

```java
...
class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    ...
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        if (ObjectUtils.nullSafeEquals("userHolder", beanName) && UserHolder.class.equals(bean.getClass())) {{

            UserHolder userHolder = (UserHolder) bean;
            userHolder.setDescription("the user holder v7");
        }}
        return bean;
    }    
}    
```

#### 2.3.2 源码讲解

```java
...
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    ...
    if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
    ...
}   
...
@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
...
```

### 2.4 初始化完成阶段

SmartInitializingSingleton#afterSingletonsInstantiated

#### 2.4.1 代码示例

```java
...
public class UserHolder implements SmartInitializingSingleton {
    ...
    @Override
    public void afterSingletonsInstantiated() {

        // postProcessAfterInitialization v7 -> init v8
        this.description = "the user holder v8";
        System.out.println("afterSingletonsInstantiated() = " + description);
    }    
}    
```

```java
...
public class BeanInitializationLifecycleDemo {

    public static void main(String[] args) {

        executeBeanFactory();
    }

    private static void executeBeanFactory() {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 方法一： 添加 BeanPostProcessor 实现（示例）
        beanFactory.addBeanPostProcessor(new MyInstantiationAwareBeanPostProcessor());
        // 添加 CommonAnnotationBeanPostProcessor 解决 @PostConstructor
        beanFactory.addBeanPostProcessor(new CommonAnnotationBeanPostProcessor());
        // 方法二：将 MyInstantiationAwareBeanPostProcessor 作为 Bean 注册
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
    }

}    
```

#### 2.4.2 源码讲解

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
    ...
	@Override
	public void preInstantiateSingletons() throws BeansException {
		...
		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
    ...
}    
```

