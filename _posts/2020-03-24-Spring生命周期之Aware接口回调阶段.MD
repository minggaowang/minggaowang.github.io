---
layout:     post
title:      Spring 生命周期之 Aware 接口回调阶段
subtitle:   Aware 接口回调阶段
date:       2020-03-24
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 Aware 接口回调阶段

它是隶属于 Bean 的 Initializing，就是所谓的 Bean 的初始化阶段。我们看到 Spring Aware 接口是一个标记接口，它没有提供任何实现，因此 BeanNameAware 以及其他 Aware 结尾都实现这个接口或者扩展这个接口。这个接口有什么意义？这个接口我们可以看出他的基本特征：Aware是个后缀，前面是它包含的含义，Aware 是一个回调，然后帮忙来进行接口的注入

## 2 Aware 接口及优先调用顺序

- BeanNameAware
- BeanClassLoaderAware
- BeamFactoryAware
- EmviromentAware
- EmbeddedValueResolverAware
- ResourceLoaderAware
- ApplicationEventPublisherAware
- MessageSourceAware
- ApplicationContextAware

## 3  Aware 接口代码实现示例

userHolder

```java
...
public class UserHolder implements BeanNameAware, BeanClassLoaderAware, BeanFactoryAware, EnvironmentAware {
	
    ...
        
    private ClassLoader classLoader;

    private BeanFactory beanFactory;

    private String beanName;

    private Environment environment;
    
    ...
        
    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {

        this.classLoader = classLoader;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {

        this.beanFactory = beanFactory;
    }

    @Override
    public void setBeanName(String name) {

        this.beanName = name;
    }

    @Override
    public void setEnvironment(Environment environment) {

        this.environment = environment;
    }
    ...
}            
```

应用实例

```java
...
public class BeanInstantiationLifecycleDemo {

    public static void main(String[] args) {

        executeBeanFactory();
        System.out.println("-----------------------------------");
        executeApplicationContext();
    }

    private static void executeBeanFactory() {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 方法一： 添加 BeanPostProcessor 实现（示例）
//        beanFactory.addBeanPostProcessor(new MyInstantiationAwareBeanPostProcessor());
        // 方法二：将 MyInstantiationAwareBeanPostProcessor 作为 Bean 注册
        // 基于 XML 资源 BeanDefinitionReader 实现
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        String[] locations = {"META-INF/dependency-lookup-context.xml",
                "META-INF/bean-constructor-dependency-injection.xml"};
        int beanNumbers = beanDefinitionReader.loadBeanDefinitions(locations);
        System.out.println("已加载 BeanDefinition 数量：" + beanNumbers);
        // 通过 Bean id 和类型进行依赖查找
        User user = beanFactory.getBean("user", User.class);
        System.out.println(user);

        User superUser = beanFactory.getBean("superUser", User.class);
        System.out.println(superUser);

        // 构造器注入按照类型注入，resolveDependency
        UserHolder userHolder = beanFactory.getBean("userHolder", UserHolder.class);
        System.out.println(userHolder);
    }

    public static void executeApplicationContext() {

        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext();
        String[] locations = {"META-INF/dependency-lookup-context.xml",
                "META-INF/bean-constructor-dependency-injection.xml"};
        applicationContext.setConfigLocations(locations);
        // 启动应用上下文
        applicationContext.refresh();
        // 构造器注入按照类型注入，resolveDependency
        UserHolder userHolder = applicationContext.getBean("userHolder", UserHolder.class);
        System.out.println(userHolder);
        // 关闭应用上下文
        applicationContext.close();
    }
}    
```



## 4 Aware 接口源码解析

BeanNameAware、BeanClassLoaderAware、BeamFactoryAware 接口调用方法入口：AbstractAutowireCapableBeanFactory#invokeAwareMethods

```java
...
private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
}
...
```

EmviromentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware 接口调用方法入口：ApplicationContextAwareProcessor#invokeAwareInterfaces

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {
    ...
    private void invokeAwareInterfaces(Object bean) {
            if (bean instanceof EnvironmentAware) {
                ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
            }
            if (bean instanceof EmbeddedValueResolverAware) {
                ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
            }
            if (bean instanceof ResourceLoaderAware) {
                ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
            }
            if (bean instanceof ApplicationEventPublisherAware) {
                ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
            }
            if (bean instanceof MessageSourceAware) {
                ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
            }
            if (bean instanceof ApplicationContextAware) {
                ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
            }
        }    
}    
```

从以上代码看出 ApplicationContextAwareProcessor 表示同包中和本类中可以引入使用，内置类。只能通过 Application 初始化的时候或者 BeanFactory 初始化的时候它会动态地到 BeanFactory 里去。而 BeanFactory 没办法从外面的方式获取，所有只能通过 ApplicationContext 方式进行回调。