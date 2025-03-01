以下是一个完整的Spring Boot自动配置实现示例，包含详细步骤和代码实现：

### 一、项目结构

```bash
my-starter/
├── my-spring-boot-autoconfigure/
│   ├── src/main/java/
│   │   └── com/example/autoconfigure/
│   │       ├── MyServiceAutoConfiguration.java   # 自动配置类
│   │       ├── MyServiceProperties.java          # 配置属性类
│   │       └── service/
│   │           └── MyService.java                # 业务服务类
│   └── src/main/resources/
│       ├── META-INF/
│       │   ├── spring.factories                  # 自动配置注册文件
│       │   └── additional-spring-configuration-metadata.json # 元数据提示
├── my-spring-boot-starter/                       # Starter模块
│   └── pom.xml
└── pom.xml
```

### 二、具体实现步骤

#### 1. 创建配置属性类

```java
// MyServiceProperties.java
@ConfigurationProperties(prefix = "my.service")
public class MyServiceProperties {
    private String prefix = "[INFO]";
    private boolean enabled = true;

    // Getter/Setter 省略...
}
```

#### 2. 创建业务服务类

```java
// MyService.java
public class MyService {
    private final String prefix;

    public MyService(String prefix) {
        this.prefix = prefix;
    }

    public String wrap(String content) {
        return prefix + " " + content;
    }
}
```

#### 3. 创建自动配置类

```java
// MyServiceAutoConfiguration.java
@Configuration
@EnableConfigurationProperties(MyServiceProperties.class)
@ConditionalOnClass(MyService.class)
@ConditionalOnProperty(prefix = "my.service", name = "enabled", havingValue = "true", matchIfMissing = true)
@AutoConfigureAfter(DataSourceAutoConfiguration.class) // 示例依赖
public class MyServiceAutoConfiguration {
    
    private final MyServiceProperties properties;

    public MyServiceAutoConfiguration(MyServiceProperties properties) {
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean
    public MyService myService() {
        return new MyService(properties.getPrefix());
    }
}
```

#### 4. 注册自动配置

```properties
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.autoconfigure.MyServiceAutoConfiguration
```

#### 5. 添加配置元数据提示

```json
# src/main/resources/META-INF/additional-spring-configuration-metadata.json
{
  "properties": [
    {
      "name": "my.service.prefix",
      "type": "java.lang.String",
      "description": "Custom prefix for service output",
      "defaultValue": "[INFO]"
    },
    {
      "name": "my.service.enabled",
      "type": "java.lang.Boolean",
      "description": "Enable/disable my service",
      "defaultValue": true
    }
  ]
}
```

### 三、创建Starter模块（可选）

```xml
<!-- my-spring-boot-starter/pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-spring-boot-autoconfigure</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

### 四、使用自动配置

```yaml
# application.yml
my:
  service:
    prefix: "[DEBUG]"
    enabled: true
```

```java
// 在Spring Boot应用中直接注入
@RestController
public class DemoController {
    private final MyService myService;

    public DemoController(MyService myService) {
        this.myService = myService;
    }

    @GetMapping("/test")
    public String test() {
        return myService.wrap("Hello World");
    }
}
```

### 五、验证配置生效

1. 启动时查看日志：

```bash
2023-09-20 10:00:00 INFO  o.s.b.a.s.s.MyServiceAutoConfiguration : MyService auto-configured with prefix: [DEBUG]
```

2. 访问端点验证：

```bash
$ curl http://localhost:8080/test
# 输出: [DEBUG] Hello World
```

### 六、测试用例

```java
@SpringBootTest
class MyServiceAutoConfigurationTest {

    @Autowired(required = false)
    private MyService myService;

    @Test
    void testAutoConfiguration() {
        assertThat(myService).isNotNull();
        assertThat(myService.wrap("test")).startsWith("[DEBUG]");
    }

    @Test
    @EnabledIfProperty(
        name = "my.service.enabled",
        havingValue = "false"
    )
    void testDisabledScenario() {
        assertThat(myService).isNull();
    }
}
```

### 七、注意事项

1. **包路径隔离**：自动配置类应该放在不会被组件扫描的独立包中
2. **条件注解顺序**：`@ConditionalOnBean` 和 `@ConditionalOnMissingBean` 要谨慎使用
3. **配置优先级**：用户自定义Bean会覆盖自动配置的Bean
4. **元数据校验**：建议添加`spring-boot-configuration-processor`依赖

完整实现需要包含以下依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```



