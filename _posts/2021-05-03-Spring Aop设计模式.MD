# SpringAop 设计模式

## 一 创建型（Creational）

### 1 抽象工厂模式（Abstract factory）

- 基本概念

  抽象工厂模式提供了一种方式，可以将一组具有同一主题的单独的工厂封装起来。在正常使用中，客户端程序需要创建抽象工厂的具体实现，然后使用抽象工厂作为接口创建这一主题的具体对象。客户端程序不需要知道（或关心）它从这些内部的工厂方法中获得对象的具体类型，因为客户端仅使用这个些对象的通用接口。抽象工厂模式将一组对象的实现细节与他们的一般使用分离开来。

- Spring AOP 举例

  - 接口 - org.springframework.aop.framework.AopProxyFactory
  - 实现 - org.springframework.aop.framework.DefaultAopProxyFactory
  - 被代理接口 - org.springframework.aop.framework.AopProxy
  - 类名开头 - 返回的类对象
  - 类名结尾 - Factory
  - 方法开头 - create

- JDK 举例

  - 接口 - java.util.concurrent.ThreadFactory
  - 实现 - org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
  - 被代理类 - java.lang.Thread
  - 类名开头 - 返回的类对象
  - 类名结尾 - Factory
  - 方法名开头 - new

### 2 构建器器模式（Builder）

- 基本概念

  建造模式，是一种对象构建模式。它可以将复制对象的建造过程抽象出来（抽象类别），使这个抽象过程的不同实现方法可以构造出不同表现（属性）的对象。

- Spring Amp 举例

  - 实现 - org.springframework.aop.aspectj.annotation.BeanFactoryAspectJAdvisorsBuilder

- JDK 举例

  - java.lang.StringBuilder - 线程不安全
  - java.lang.StringBuffer - 线程安全

### 3 工厂方法模式（Factory method）

- 基本概念

  就像其他创建模式一样，它也是处理在不指定对象具体类型的情况下创建对象的问题。工厂方法模式的实质是"定义一个创建对象的接口，但让实现这个接口的类来决定实例化哪个类。工厂方法让类的实例化推迟到子类中进行。"

- Spring AOP 举例

  - 实现 - org.springframework.aop.framework.ProxyFactory
  - 方法（动态）- org.springframework.aop.framework.ProxyFactory#getProxy()
  - 方法（静态） - org.springframework.aop.framework.ProxyFactory#getProxy(java.lang.Class<T>, org.aopalliance.intercept.Interceptor)

### 4 原型模式（Prototype）

- 基本概念

  创建型模式的一种，其特点在于通过"复制"一个已经存在的实例来返回新的实例，而不是新建实例。被复制的实例就是我们所称的原型，这个原型是可定制的。

  原型模式多用于创建复杂的或者耗时的实例，因为这中情况下，复制一个已经存在的实例使程序运行更高效；或者创建值相等，只是命名不一样的同类数据。

- Spring AOP 举例

  - 实现 - org.springframework.aop.target.PrototypeTargetSource

### 5 单例模式（Singleton）

- 基本概念

  属于创建模式的一种。在应用这个模式时，单例对象的类必须保证只有一个实例存在。许多时候整个系统只需要拥有一个的全局对象，这样有利于我们协调系统整体的行为。比如在某个服务器程序中，该服务器的配置信息存放在一个文件中，这些配置数据由一个单例对象统一读取，然后服务进程中的其他对象再通过这个单例对象获取这些配置信息。这种方法简化了在复杂环境下的配置管理。

- Spring Aop 举例

  - 实现 - org.springframework.aop.target.SingletonTargetSource

- Spring Ioc 举例

  - 实现 - org.springframework.core.convert.support.DefaultConversionService
  - 实现 - org.springframework.aop.framework.adapter.GlobalAdvisorAdapterRegistry

## 二 结构型（Structural）

### 1 适配器模式（Adapter）

- 基本概念

  有时候也称包装样式或者包装（wrapper）。将一个类的接口转接成用户所期待的。一个适配使得因接口不兼容而不能在一起工作的类能在一起工作，做法是将类自己的接口包裹在一个已存在的类中。

- Spring Aop 举例

  - 接口 - org.springframework.aop.framework.adapter.AdvisorAdapter
  - 适配对象 - org.aopalliance.aop.Advice
  - 目标对象 - org.aopalliance.intercept.MethodInterceptor
  - 以下二合一
    - org.springframework.core.convert.converter.Converter
    - org.springframework.core.convert.converter.ConditionalConverter

### 2 组合模式（Composite）

- 基本概念

  把一组对象当成一个对象来处理，他们之间是相同类型的。

- Spring Aop 举例

  - 实现 - org.springframework.aop.support.ComposablePointcut
  - 接口 - org.springframework.aop.Pointcut
  - 成员 - org.springframework.aop.Pointcut

- Spring Ioc 举例

  - 实现 - org.springframework.core.env.MutablePropertySources	
  - 成员 - org.springframework.core.env.PropertySource

### 3 装饰器模式（Decorator）

- 基本概念

  一种动态地往一个类中添加新的行为的设计模式。就功能而言，修饰模式相比生成子类更为灵活，这样可以给某个对象而不是整个类添加一些功能。

- Spring AOP 举例

  - 实现 - org.springframework.aop.aspectj.annotation.LazySingletonAspectInstanceFactoryDecorator

- JDK 举例

  - 实现 - java.io.BufferedInputStream

- Servlet 举例

  - 实现 - javax.servlet.ServletRequestWrapper

### 4 享元模式（Flyweight）

- 基本概念

  他使用物件用来尽可能减少记忆体使用量；于相似物件中分享尽可能多的资讯。当大量物件近乎重复方式存在，因而使用大量记忆体时，此法通过。通常物件中的部分状体能够共享。常见做法是他们放在资料结构外部，当需要使用时再将它们传递给享元。

- Spring Aop 举例

  - 接口 - org.springframework.aop.framework.adapter.AdvisorAdapterRegistry
  - 实现类 - org.springframework.aop.framework.adapter.DefaultAdvisorAdapterRegistry
  - 调用类 - org.springframework.aop.framework.adapter.GlobalAdvisorAdapterRegistry

- Spring Ioc 举例

  - 实现 - org.springframework.core.convert.support.DefaultConversionService

### 5 代理模式（Proxy）

- 基本概念

  所谓的代理者是指一个类别可以作为其它东西的介面。代理者可以作任何东西的介面：网路连接、记忆体中的大物件、档案或其它昂贵或无法复制的资源。

- Spring Aop 举例

  - 实现 - org.springframework.aop.framework.AopProxy
    - JDK - org.springframework.aop.framework.JdkDynamicAopProxy
    - CGLIB - org.springframework.aop.framework.CglibAopProxy

### 6 模版方法模式（Template Method）

- 基本概念

  模版方法是一个定义在父类的方法，在模版方法中会呼叫多个定义在父类的其他方法，而这些方法有可能只是抽象方法并没有实作，模版方法仅决定这些抽象方法的执行顺序，这些抽象方法的实作由子类负责，并且子类不允许复写模版方法。

- Spring Aop 举例

  - 模版类 - org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
  - 模版方法- org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#getAdvicesAndAdvisorsForBean
  - 子类实现
    - XML实现 - org.springframework.aop.framework.autoproxy.InfrastructureAdvisorAutoProxyCreator
    - 注解实现 - org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator

## 三 行为型（Behavioral）

### 1 责任链模式（Chain of Responsibility）

- 基本概念

  包含了一些命令对象和一系列的处理对象。每一个处理对象决定它能处理那些命令对象，它也知道如果将它不能处理的命令对象传递给该链中的下一个处理对象。该模式还描述了往该处理链的末尾添加新的处理对象的方法。

- Spring Aop 举例

  - 接口 - org.springframework.aop.framework.AdvisorChainFactory
  - 实现 - org.springframework.aop.framework.DefaultAdvisorChainFactory

### 2 观察者模式（Observer）

- 基本概念

  一个目标物件管理所有相依于它的观察者物件，并且在它本身的状态能改变时主动发出通知。这样通常透过呼叫各观察者所提供的方法来实现。此中模式通常被用来实时事件处理系统。

- Spring Aop 举例

  - 观察者 - org.springframework.aop.framework.ProxyCreatorSupport
  - 被观察者 - org.springframework.aop.framework.AdvisedSupportListener
  - 通知对象 - org.springframework.aop.framework.AdvisedSupport

### 3 策略模式（Strategy）

- 基本概念

  指对象有某个行为，但是在不同的场景中，该行为有不同的实现算法。比如每个人都要“交个人所得税“，但是每国交个人所得税就有不同的算税方法。

- Spring Aop 举例

  - 实现方法 - org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy
  - 实现方法 - org.springframework.aop.config.ConfigBeanDefinitionParser#getAdviceClass

### 4 命令模式（Command）

- 基本概念

  它尝试以物件来代表实际行动。命令物件可以把行动及其参数封装起来，于是这些行动可以被：

  — 重复多次

  —取消（如果该物件实作的话）

  —取消后又再重做

- Spring Aop 举例

  - org.aopalliance.intercept.MethodInvocation

    - org.springframework.aop.framework.CglibAopProxy.CglibMethodInvocation#proceed

    - org.springframework.aop.framework.JdkDynamicAopProxy#invoke

      ```java
      ...
      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
          // We can skip creating a MethodInvocation: just invoke the target directly
          // Note that the final invoker must be an InvokerInterceptor so we know it does
          // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
          Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
          retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }else {
        // We need to create a method invocation...
        MethodInvocation invocation =
          new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
        // Proceed to the joinpoint through the interceptor chain.
        retVal = invocation.proceed();
      }
      ...

  - org.aspectj.lang.ProceedingJoinPoint

- JDK 举例

  - java.lang.Runnable
  - java.util.concurrent.Callable
  - java.lang.reflect.InvocationHandler

### 5 状态模式（State）

- 基本概念

  允许对象在内部发生变化时更改其行为。这种模式接近于有限状态机的概念。状态模式可以解释为策略模式，它能够通过调用模式接口定义的方法来切换策略。

- Spring Aop 举例

  - 状态对象 - org.springframework.aop.framework.ProxyConfig
  - 影响对象 - org.springframework.aop.framework.AopProxy
  - org.springframework.aop.framework.JdkDynamicAopProxy
  - org.springframework.aop.framework.CglibAopProxy

