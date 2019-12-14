---
layout:     post
title:      SpringIOC扫描bean定义源码解析
subtitle:   SpringIOC
date:       2019-12-14
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 分析之前的相关知识

### 1.1 ApplicationListener事件发布

> 写一个 DemoApplicationListener 实现 ApplicationListener 接口，并且把该组件加入到容器中。

```java
...
@Component
public class DemoApplicationListener implements ApplicationListener {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("DemoApplicationListener 接受到了一个事件"+event);
    }
}
```

> 扫描包

```java
...
@ComponentScan(basePackages = {"com.demo.testapplicationlistener"})
public class MainConfig {
}
```

> 如果使用的默认的广播器，则直接发布时间。

```java
...
public class MainClass {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
        ctx.publishEvent(new ApplicationEvent("我手动发布了一个事件") {
            @Override
            public Object getSource() {
                return super.getSource();
            }
        });

        ctx.close();
    }
}
```

> 通过继承 SimpleApplicationEventMulticaster 定义广播器，实现广播器支持异步。

```java
...
@Component(value = "applicationEventMulticaster")
public class DemoMulticaster extends SimpleApplicationEventMulticaster{

    public TulingMulticaster () {
        setTaskExecutor(Executors.newSingleThreadExecutor());
    }

}    
```

>可以通过 BeanPostProcessor Bean的后置处理器，实现广播器支持异步。

```java
...
@Component
public class MultiCasterBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof SimpleApplicationEventMulticaster) {
            System.out.println("我拦截到了");
            SimpleApplicationEventMulticaster simpleApplicationEventMulticaster= (SimpleApplicationEventMulticaster) bean;
            simpleApplicationEventMulticaster.setTaskExecutor(Executors.newSingleThreadExecutor());
            return bean;
        }
        return bean;

    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

### 1.2 Bean的后置处理器（BeanPostProcessor）

```java
// 见如上MultiCasterBeanPostProcessor
...
```

### 1.3 BeanFactory的后置处理器（BeanFactoryPostProcessor）

```java
...
@Component
public class DemoBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("IOC 容器调用了DemoBeanFactoryPostProcessor的postProcessBeanFactory方法");
        for(String name:beanFactory.getBeanDefinitionNames()) {
            if("tulingLog".equals(name)) {
                //从 BeanFactory 中获取BeanDefinition(Bean的定义类)
                BeanDefinition beanDefinition = beanFactory.getBeanDefinition(name);
                //将 Bean 定义为懒加载，当 Bean 第一次使用的时候实例化到容量。
                beanDefinition.setLazyInit(true);
            }

        }
    }
}
```

### 1.4 BeanDefinationRegister的后置处理器（BeanDefinitionRegistryPostProcessor）

```java
...
@Component
public class DemoBeanDefinationRegisterPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        System.out.println("DemoBeanDefinationRegisterPostProcessor的postProcessBeanDefinitionRegistry方法");
        System.out.println("bean定义的数据量:"+registry.getBeanDefinitionCount());
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(DemoLog.class);
        //将DemoLog Bean定义注册到容器中
        registry.registerBeanDefinition("demoLog",rootBeanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("DemoBeanDefinationRegisterPostProcessor的postProcessBeanFactory方法");
        System.out.println(beanFactory.getBeanDefinitionCount());
    }
}    
```

## 2 源码分析

### 2.1 main 类入口

```java
public class MainClass {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
        ctx.close();
    }
}		
```

> 扫描包

```java
@Configuration
@ComponentScan(basePackages = "com.demo.testbeanfacotoryPostProcessor")
public class MainConfig {

}
```

### 2.2 创建 AnnotationConfigApplicationContext 构造方法

```java
...
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
        // 注册内部类添加到容器中(beanDefinitionMap，beanDefinitionNames)
		// org.springframework.context.annotation.internalConfigurationAnnotationProcessor
    	// org.springframework.context.event.internalEventListenerFactory
        // org.springframework.context.event.internalEventListenerProcessor
    	// org.springframework.context.annotation.internalAutowiredAnnotationProcessor
        // org.springframework.context.annotation.internalRequiredAnnotationProcessor
        this();
        // 注册外部的注解类添加到容器中(beanDefinitionMap，beanDefinitionNames)
        // mainConfig
		register(annotatedClasses);
        // 重点分析
        // bean生命周期
		refresh();
	}
...
```

### 2.3 调用 org.springframework.context.support.AbstractApplicationContext#refresh

```java
...
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // 重点分析
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
...
```

### 2.4 调用 PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(beanFactory, beanFactoryPostProcessors)

```java
...
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    ...
// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
    		//重点分析
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
    ...    
}    
```

### 2.5 调用 PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors(postProcessors, registry)

```java
/**
  * Invoke the given BeanDefinitionRegistryPostProcessor beans.
  */
private static void invokeBeanDefinitionRegistryPostProcessors(
    Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

    for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
        // 重点分析
        postProcessor.postProcessBeanDefinitionRegistry(registry);
    }
}
```

### 2.6 调用 ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry(registry)

```java
...
/**
  * Derive further bean definitions from the configuration classes in the registry.
  */
@Override
public void postProcessBeanDefinitionRegistry(registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);

    processConfigBeanDefinitions(registry);
}
...
```

### 2.7 调用 ConfigurationClassPostProcessor#processConfigBeanDefinitions(registry)

```java
/**
  * Build and validate a configuration model based on the registry of
  * {@link Configuration} classes.
  */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
       //第一步 过滤我们传入的主配置类
        List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
					ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		} 
    
    //第二步 创建一个配置类解析器(ConfigurationClassParser)
    // Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());
		do {
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<String>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (sbr != null) {
			if (!sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
				sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
			}
		}

		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
}
```

> 见下图

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring1.jpg)