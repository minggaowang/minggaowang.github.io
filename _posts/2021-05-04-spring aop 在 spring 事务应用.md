# spring aop 在 spring 事务应用

- 核心 API
  - Spring 事务 @Enable 模块驱动 - @EnableTransactionManagement
  - Spring 事务注解 - @Transactional
  - Spring 事务事件监听器 - @TransactionalEventListener
  - Spring 事务定义 - TransactionDefinition
  - Spring 事务状态 - TransactionStatus
  - Spring 平台事务管理器 - PlatformTransactionManager
  - Spring 事务代理配置 - ProxyTransactionManagementConfiguration
  - Spring 事务 PointcutAdvisor 实现 - BeanFactoryTransactionAttributeSourceAdvisor
  - Spring 事务 MethodInterceptor 实现 - TransactionInterceptor
  - Spring 事务属性源 - TransactionAttributeSource
- 理解 TransactionDefinition（Spring 事务定义）
  - 说明：定义与Spring兼容的事务属性的接口。 基于类似于EJB CMT属性的传播行为定义。
  - 核心方法
    - getIsolationLevel()：获取隔离级别，默认值 ISOLATION_DEFAULT 常量，参考org.springframework.transaction.annotation.Isolation
    - getPropagationBehavior()：获取事务传播，默认值 PROPAGATION_REQUIRED 常量，参考org.springframework.transaction.annotation.Propagation
    - getTimeout()：获取事务执行超时时间，默认值：TIMEOUT_DEFAULT 常量
    - isReadOnly()：是否为只读事务，默认值：false
- 理解 TransactionStatus（Spring 事务状态）
  - 说明：交易状态的表示。事务代码可以使用它来检索状态信息，并以编程方式请求回滚（而不是引发导致隐式回滚的异常）。包括 SavepointManager 界面，以提供对保存点管理工具的访问。 请注意，只有在基础事务管理器支持的情况下，保存点管理才可用。
  - 核心方法
    - hasSavepoint()：是否已基于保存点将其创建为嵌套事务。
    - flush()：如果适用，将基础会话刷新到数据存储区：例如，所有受影响的Hibernate / JPA会话
    - isNewTransaction()：当前事务执行是否新的事务
    - setRollbackOnly()：将当前事务设置为只读
    - isRollbackOnly()：当前事务是否为只读
    - isCompleted()：当前事务是否完成
- 理解 PlatformTransactionManager（Spring 平台事务管理器）
  - 说明：这是Spring事务基础架构中的中央接口。 应用程序可以直接使用它，但是它并不是主要用于API：通常，应用程序可以通过Transaction模板或通过AOP进行声明式事务划分来使用。
  - 核心方法
    - getTransaction(TransactionDefinition definition)：获取事务状态
    - commit(TransactionStatus status)：提交事务
    - rollback(TransactionStatus status)：回滚事务
- 理解 Spring 事务传播（Transaction Propagation）
  - 官方文档：https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-propagation

