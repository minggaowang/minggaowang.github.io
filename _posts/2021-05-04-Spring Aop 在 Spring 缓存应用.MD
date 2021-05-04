# Spring Aop 在 Spring 缓存应用（Caching）

- 核心 API
  - Spring 缓存 @Enable 模块驱动 - @EnableCaching
  - 缓存操作注解 - @Caching、@Cacheable、@CachePut、@CacheEvict
  - 缓存配置注解 - @CacheConfig
  - 缓存注解操作数据源 - AnnotationCacheOperationSource
  - Spring 缓存注解解析器 - SpringCacheAnnotationParser
  - Spring 缓存管理器 - CacheManager
  - Spring 缓存接口 - Cache
  - Spring 缓存代理配置 - ProxyCachingConfiguration
  - Spring 缓存 PointcutAdvisor 实现 - BeanFactoryCacheOperationSourceAdvisor
  - Spring 缓存 MethodInterceptor 实现 - CacheInterceptor