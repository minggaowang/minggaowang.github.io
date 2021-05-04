# spring aop 在 spring 事件应用

- 核心 API - org.springframework.context.event.EventPublicationInterceptor

- 特性描述

  当 Spring AOP 代理 Bean 中的 JoinPoint 方法执行之后，Spring ApplicationContext 将发布一个自定义事件（ApplicationEvent 子类）

- 使用限制

  - EventPublicationInterceptor 关联的 ApplicationEvent 子类必须存在参数的构造器
  - EventPublicationInterceptor 需要被声明为 Spring Bean

- 实例代码

  - 执行事件类

  ```java
  public class ExecutedEvent extends ApplicationEvent {
  
      /**
       * Create a new {@code ApplicationEvent}.
       *
       * @param source the object on which the event initially occurred or with
       *               which the event is associated (never {@code null})
       */
      public ExecutedEvent(Object source) {
          super(source);
      }
  }
  ```

  - 执行器

  ```java
  public class Executor { // ClassFilter
  
      public void execute() { // MethodMatcher: Join Point 方法（需要 Pointcut 来匹配）
          System.out.println("Executing...");
      }
  }
  ```

  - 静态执行器 - 传统示例

  ```java
  public class StaticExecutor implements ApplicationEventPublisherAware {
  
      private ApplicationEventPublisher applicationEventPublisher;
  
      public void execute() {
          System.out.println("Executing...");
          applicationEventPublisher.publishEvent(new ExecutedEvent(this));
      }
  
      @Override
      public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
          this.applicationEventPublisher = applicationEventPublisher;
      }
  }
  ```

  - EventPublicationInterceptor 示例

  ```java
  @Configuration // Configuration Class
  @EnableAspectJAutoProxy
  public class EventPublicationInterceptorDemo {
  
      public static void main(String[] args) {
          AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
          context.register(EventPublicationInterceptorDemo.class, Executor.class, StaticExecutor.class);
          // 启动 Spring 应用上下文
          context.refresh();
  
          // 5. 执行目标方法
          Executor executor = context.getBean(Executor.class);
  
          StaticExecutor staticExecutor = context.getBean(StaticExecutor.class);
  
          executor.execute();
  
          staticExecutor.execute();
  
          // 关闭 Spring 应用上下文
          context.close();
      }
  
      // 1. 将 EventPublicationInterceptor 声明为 Spring Bean
      @Bean
      public static EventPublicationInterceptor eventPublicationInterceptor() {
          EventPublicationInterceptor eventPublicationInterceptor = new EventPublicationInterceptor();
          // 关联目标（自定义）事件类型 - ExecutedEvent
          eventPublicationInterceptor.setApplicationEventClass(ExecutedEvent.class);
          return eventPublicationInterceptor;
      }
  
      // 2. 实现 Pointcut Bean
      @Bean
      public static Pointcut pointcut() {
          return new StaticMethodMatcherPointcut() {
              @Override
              public boolean matches(Method method, Class<?> targetClass) {
                  return "execute".equals(method.getName()) && Executor.class.equals(targetClass);
              }
          };
      }
  
      // 3. 声明一个 Advisor Bean
      @Bean
      public static PointcutAdvisor pointcutAdvisor(Pointcut pointcut, EventPublicationInterceptor eventPublicationInterceptor) {
          // EventPublicationInterceptor is MethodInterceptor is Advice
          return new DefaultPointcutAdvisor(pointcut, eventPublicationInterceptor);
      }
  
      // 4. 处理事件 - ExecutedEvent
      @EventListener(ExecutedEvent.class)
      public void executed(ExecutedEvent event) {
          System.out.println("Executed : " + event);
      }
  }
  ```

  