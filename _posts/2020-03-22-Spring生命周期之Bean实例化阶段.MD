---
layout:     post
title:      Spring 生命周期之 Bean 实例化阶段
subtitle:   Bean 实例化阶段
date:       2020-03-22
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - Spring
---

![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/spring.jpg)

## 1 Bean 实例化阶段

我们之前了解到 BeanDefinition 里面的 Class 信息，从过去的一个文本信息变成了实在的 Class 对象，这个过程被称为 Definition 或者 Bean 的 Class Loading，也称为类加载。接下来我们会将其类加载，变成一个 Bean 实例，我们叫做实例化阶段。

## 2 Bean 实例化阶段可以细分为以下阶段

- 实例化前阶段
  - InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation
- 实例化中阶段
  - 通过 JAVA 反射机制，即有参或者无参构造函数构建。
- 实例化后阶段
  - InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation
- 实例化属性赋值
  - InstantiationAwareBeanPostProcessor#postProcessProperties

## 3 Bean 实例化阶段（源码分析）

### 3.1 实例化前阶段

前阶段方法调用入口：AbstractAutowireCapableBeanFactory#createBean

```java
...
//---------------------------------------------------------------------
	// Implementation of relevant AbstractBeanFactory template methods
	//---------------------------------------------------------------------

	/**
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 * @see #doCreateBean
	 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
...
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
...
}
...
```

### 3.2 实例化中阶段

方法调用入口：AbstractAutowireCapableBeanFactory#createBean

```java
...
//---------------------------------------------------------------------
	// Implementation of relevant AbstractBeanFactory template methods
	//---------------------------------------------------------------------

	/**
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 * @see #doCreateBean
	 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
...
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
...
}
...
```

调用方法：AbstractAutowireCapableBeanFactory#doCreateBean

```java
...
/**
	 * Actually create the specified bean. Pre-creation processing has already happened
	 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
	 * <p>Differentiates between default bean instantiation, use of a
	 * factory method, and autowiring a constructor.
	 * @param beanName the name of the bean
	 * @param mbd the merged bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a new instance of the bean
	 * @throws BeanCreationException if the bean could not be created
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
 	...
     if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	 }
 	...    
   }
}    
```

调用方法：AbstractAutowireCapableBeanFactory#createBeanInstance

```java
/**
	 * Create a new instance for the specified bean, using an appropriate instantiation strategy:
	 * factory method, constructor autowiring, or simple instantiation.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a BeanWrapper for the new instance
	 * @see #obtainFromSupplier
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 * @see #instantiateBean
	 */
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
        // Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		return instantiateBean(beanName, mbd);
    }
```

调用方法：AbstractAutowireCapableBeanFactory#instantiateBean

```java
/**
	 * Instantiate the given bean using its default constructor.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @return a BeanWrapper for the new instance
	 */
	protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
```

调用方法：SimpleInstantiationStrategy#instantiate

```java
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
							constructorToUse = clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```

调用方法：BeanUtils#instantiateClass

```java
/**
	 * Convenience method to instantiate a class using the given constructor.
	 * <p>Note that this method tries to set the constructor accessible if given a
	 * non-accessible (that is, non-public) constructor, and supports Kotlin classes
	 * with optional parameters and default values.
	 * @param ctor the constructor to instantiate
	 * @param args the constructor arguments to apply (use {@code null} for an unspecified
	 * parameter, Kotlin optional parameters and Java primitive types are supported)
	 * @return the new instance
	 * @throws BeanInstantiationException if the bean cannot be instantiated
	 * @see Constructor#newInstance
	 */
	public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
		Assert.notNull(ctor, "Constructor must not be null");
		try {
			ReflectionUtils.makeAccessible(ctor);
			if (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
				return KotlinDelegate.instantiateClass(ctor, args);
			}
			else {
				Class<?>[] parameterTypes = ctor.getParameterTypes();
				Assert.isTrue(args.length <= parameterTypes.length, "Can't specify more arguments than constructor parameters");
				Object[] argsWithDefaultValues = new Object[args.length];
				for (int i = 0 ; i < args.length; i++) {
					if (args[i] == null) {
						Class<?> parameterType = parameterTypes[i];
						argsWithDefaultValues[i] = (parameterType.isPrimitive() ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
					}
					else {
						argsWithDefaultValues[i] = args[i];
					}
				}
				return ctor.newInstance(argsWithDefaultValues);
			}
		}
		catch (InstantiationException ex) {
			throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
		}
		catch (IllegalAccessException ex) {
			throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
		}
		catch (IllegalArgumentException ex) {
			throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
		}
		catch (InvocationTargetException ex) {
			throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
		}
	}
```

### 3.3 实例化后阶段

方法调用入口：AbstractAutowireCapableBeanFactory#createBean（源码如上）

调用方法：AbstractAutowireCapableBeanFactory#doCreateBean

```java
...
/**
	 * Actually create the specified bean. Pre-creation processing has already happened
	 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
	 * <p>Differentiates between default bean instantiation, use of a
	 * factory method, and autowiring a constructor.
	 * @param beanName the name of the bean
	 * @param mbd the merged bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a new instance of the bean
	 * @throws BeanCreationException if the bean could not be created
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
        ...
            populateBean(beanName, mbd, instanceWrapper);
        ...    
    }
...
```

调用方法：AbstractAutowireCapableBeanFactory#populateBean

```java
...
/**
	 * Populate the bean instance in the given BeanWrapper with the property values
	 * from the bean definition.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw the BeanWrapper with bean instance
	 */
	@SuppressWarnings("deprecation")  // for postProcessPropertyValues
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    ...
        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
    ...    
}
...
```



### 3.4 实例化属性赋值

方法调用入口：AbstractAutowireCapableBeanFactory#createBean（源码如上）

调用方法：AbstractAutowireCapableBeanFactory#doCreateBean（源码如上）

调用方法：AbstractAutowireCapableBeanFactory#populateBean

```java
...
/**
	 * Populate the bean instance in the given BeanWrapper with the property values
	 * from the bean definition.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw the BeanWrapper with bean instance
	 */
	@SuppressWarnings("deprecation")  // for postProcessPropertyValues
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
       ...
       pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, 
       ...
    }
...
```

## 4 Bean 实例化各阶段接口回调示例

```java
...
public class BeanInstantiationLifecycleDemo {

    public static void main(String[] args) {

        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 添加 BeanPostProcessor 实现（示例）
        beanFactory.addBeanPostProcessor(new MyInstantiationAwareBeanPostProcessor());
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

    static class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

        @Override
        public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {

            if (ObjectUtils.nullSafeEquals("superUser", beanName) && SuperUser.class.equals(beanClass)) {
                // 把配置完成 superUser Bean 覆盖
                return new SuperUser();
            }
            return null; // 保持 Spring IOC 容器的实例化操作
        }

        @Override
        public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {

            if (ObjectUtils.nullSafeEquals("user", beanName) && User.class.equals(bean.getClass())) {
                User user = (User) bean;
                user.setId(2L);
                user.setName("niangao");
                // user 对象不允许属性赋值（配置元信息 -> 属性值）
                return false;
            }
            return true;
        }

        // user 是跳过 Bean 属性赋值（填入）
        // superUser 也是完全跳过 Bean 实例化（Bean 属性赋值（填入））
        // userHolder
        @Override
        public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {

            // 对 userHolder Bean 进行拦截
            if (ObjectUtils.nullSafeEquals("userHolder", beanName) && UserHolder.class.equals(bean.getClass())) {


                MutablePropertyValues propertyValues;
                if (pvs instanceof MutablePropertyValues) {
                    propertyValues = (MutablePropertyValues) pvs;
                } else {
                    propertyValues = new MutablePropertyValues();
                }
                propertyValues.addPropertyValue("number", "1");

                if (propertyValues.contains("description")) {
                    // propertyValue value 是不可变的
//                    PropertyValue propertyValue = propertyValues.getPropertyValue("description");
                    propertyValues.removePropertyValue("description");
                    propertyValues.addPropertyValue("description", "this user holder V2");
                }
                return propertyValues;
            }
            return null;
        }
    }
}
...
```