在Spring Boot中实现WebSocket多实例时，由于WebSocket连接是有状态的，多个实例之间需要共享连接信息。通常可以通过以下两种方式实现：

### 1. 使用消息中间件（如RabbitMQ、Kafka）

通过消息中间件，多个实例可以共享消息，确保客户端无论连接到哪个实例都能收到消息。

#### 实现步骤：

1. **引入依赖**：
   在`pom.xml`中添加WebSocket和消息中间件（如RabbitMQ）的依赖：
   
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-websocket</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-amqp</artifactId>
   </dependency>
   ```
2. **配置WebSocket**：
   配置WebSocket端点：
   
   ```java
   @Configuration
   @EnableWebSocketMessageBroker
   public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
   
       @Override
       public void configureMessageBroker(MessageBrokerRegistry config) {
           config.enableStompBrokerRelay("/topic")
                 .setRelayHost("localhost")
                 .setRelayPort(61613)
                 .setClientLogin("guest")
                 .setClientPasscode("guest");
           config.setApplicationDestinationPrefixes("/app");
       }
   
       @Override
       public void registerStompEndpoints(StompEndpointRegistry registry) {
           registry.addEndpoint("/ws").withSockJS();
       }
   }
   ```
3. **消息生产者**：
   使用`SimpMessagingTemplate`发送消息：
   
   ```java
   @Autowired
   private SimpMessagingTemplate template;
   
   public void sendMessage(String message) {
       template.convertAndSend("/topic/messages", message);
   }
   ```
4. **消息消费者**：
   监听消息中间件的消息并转发给客户端：
   
   ```java
   @Component
   public class MessageListener {
   
       @Autowired
       private SimpMessagingTemplate template;
   
       @RabbitListener(queues = "websocket-queue")
       public void receiveMessage(String message) {
           template.convertAndSend("/topic/messages", message);
       }
   }
   ```
5. **配置RabbitMQ**：
   在`application.properties`中配置RabbitMQ：
   
   ```properties
   spring.rabbitmq.host=localhost
   spring.rabbitmq.port=5672
   spring.rabbitmq.username=guest
   spring.rabbitmq.password=guest
   ```

### 2. 使用Redis Pub/Sub

Redis的发布/订阅功能也可以用于多实例间的消息共享。

#### 实现步骤：

1. **引入依赖**：
   在`pom.xml`中添加WebSocket和Redis的依赖：
   
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-websocket</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-redis</artifactId>
   </dependency>
   ```
2. **配置WebSocket**：
   配置WebSocket端点：
   
   ```java
   @Configuration
   @EnableWebSocketMessageBroker
   public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
   
       @Override
       public void configureMessageBroker(MessageBrokerRegistry config) {
           config.enableSimpleBroker("/topic");
           config.setApplicationDestinationPrefixes("/app");
       }
   
       @Override
       public void registerStompEndpoints(StompEndpointRegistry registry) {
           registry.addEndpoint("/ws").withSockJS();
       }
   }
   ```
3. **消息生产者**：
   使用`SimpMessagingTemplate`发送消息：
   
   ```java
   @Autowired
   private SimpMessagingTemplate template;
   
   public void sendMessage(String message) {
       template.convertAndSend("/topic/messages", message);
   }
   ```
4. **消息消费者**：
   监听Redis的频道并转发消息：
   
   ```java
   @Component
   public class RedisMessageListener {
   
       @Autowired
       private SimpMessagingTemplate template;
   
       @Autowired
       private StringRedisTemplate redisTemplate;
   
       @PostConstruct
       public void init() {
           redisTemplate.getConnectionFactory().getConnection().subscribe((message, pattern) -> {
               String receivedMessage = new String(message.getBody());
               template.convertAndSend("/topic/messages", receivedMessage);
           }, "websocket-channel".getBytes());
       }
   }
   ```
5. **配置Redis**：
   在`application.properties`中配置Redis：
   
   ```properties
   spring.redis.host=localhost
   spring.redis.port=6379
   ```

### 总结

- **消息中间件**：适合需要高可靠性和复杂路由的场景。
- **Redis Pub/Sub**：适合轻量级、低延迟的场景。

根据需求选择合适的方案即可。

