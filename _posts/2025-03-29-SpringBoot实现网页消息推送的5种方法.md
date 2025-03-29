# SpringBoot实现网页消息推送的5种方法

## 一、为什么需要消息推送？

传统的HTTP请求是客户端主动请求，服务端被动响应的模式。但在很多场景下，我们需要服务器能够主动将消息推送给浏览器，例如：

+ Web版即时通讯
+ 股票、基金等金融数据实时更新
+ 系统通知和提醒
+ 协同编辑文档时的实时更新
+ ......

## 二、消息推送实现方案

### 1\. 短轮询 (Short Polling)

原理：客户端以固定的时间间隔频繁发送请求，询问服务器是否有新消息。

实现方式：

```less
@RestController
@RequestMapping("/api/messages")
public class MessageController {
    
    private final Map<String, List<String>> userMessages = new ConcurrentHashMap<>();
    
    @GetMapping("/{userId}")
    public List<String> getMessages(@PathVariable String userId) {
        List<String> messages = userMessages.getOrDefault(userId, new ArrayList<>());
        List<String> result = new ArrayList<>(messages);
        messages.clear();  // 清空已读消息
        return result;
    }
    
    @PostMapping("/{userId}")
    public void sendMessage(@PathVariable String userId, @RequestBody String message) {
        userMessages.computeIfAbsent(userId, k -> new ArrayList<>()).add(message);
    }
}
```

前端实现：

```javascript
function startPolling() {
    setInterval(() => {
        fetch('/api/messages/user123')
            .then(response => response.json())
            .then(messages => {
                if (messages.length > 0) {
                    messages.forEach(msg => console.log(msg));
                }
            });
    }, 3000); // 每3秒查询一次
}
```

优点：

+ 实现简单，不需要特殊的服务器配置
+ 兼容性好，支持几乎所有浏览器和服务器

缺点：

+ 资源消耗大，大量无效请求
+ 实时性较差，受轮询间隔影响
+ 服务器负载高，尤其是在用户量大的情况下

### 2\. 长轮询 (Long Polling)

原理：客户端发送请求后，如果服务器没有新消息，则保持连接打开直到有新消息或超时，然后客户端立即发起新的请求。

实现方式：

```less
@RestController
@RequestMapping("/api/long-polling")
public class LongPollingController {
    
    private final Map<String, DeferredResult<List<String>>> waitingRequests = new ConcurrentHashMap<>();
    private final Map<String, List<String>> pendingMessages = new ConcurrentHashMap<>();
    
    @GetMapping("/{userId}")
    public DeferredResult<List<String>> waitForMessages(@PathVariable String userId) {
        DeferredResult<List<String>> result = new DeferredResult<>(60000L, new ArrayList<>());
        
        // 检查是否有待处理的消息
        List<String> messages = pendingMessages.get(userId);
        if (messages != null && !messages.isEmpty()) {
            List<String> messagesToSend = new ArrayList<>(messages);
            messages.clear();
            result.setResult(messagesToSend);
        } else {
            // 没有消息，等待
            waitingRequests.put(userId, result);
            
            result.onCompletion(() -> waitingRequests.remove(userId));
            result.onTimeout(() -> waitingRequests.remove(userId));
        }
        
        return result;
    }
    
    @PostMapping("/{userId}")
    public void sendMessage(@PathVariable String userId, @RequestBody String message) {
        // 查看是否有等待的请求
        DeferredResult<List<String>> deferredResult = waitingRequests.get(userId);
        
        if (deferredResult != null) {
            List<String> messages = new ArrayList<>();
            messages.add(message);
            deferredResult.setResult(messages);
            waitingRequests.remove(userId);
        } else {
            // 存储消息，等待下一次轮询
            pendingMessages.computeIfAbsent(userId, k -> new ArrayList<>()).add(message);
        }
    }
}
```

前端实现：

```scss
function longPolling() {
    fetch('/api/long-polling/user123')
        .then(response => response.json())
        .then(messages => {
            if (messages.length > 0) {
                messages.forEach(msg => console.log(msg));
            }
            // 立即发起下一次长轮询
            longPolling();
        })
        .catch(() => {
            // 出错后延迟一下再重试
            setTimeout(longPolling, 5000);
        });
}
```

优点：

+ 减少无效请求，相比短轮询更高效
+ 近实时体验，有消息时立即推送
+ 兼容性好，几乎所有浏览器都支持

缺点：

+ 服务器资源消耗，大量连接会占用服务器资源
+ 可能受超时限制
+ 难以处理服务器主动推送的场景

### 3\. Server-Sent Events (SSE)

原理：服务器与客户端建立单向连接，服务器可以持续向客户端推送数据，而不需要客户端重复请求。

SpringBoot实现：

```less
@RestController
@RequestMapping("/api/sse")
public class SSEController {
    
    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();
    
    @GetMapping("/subscribe/{userId}")
    public SseEmitter subscribe(@PathVariable String userId) {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
        
        emitter.onCompletion(() -> emitters.remove(userId));
        emitter.onTimeout(() -> emitters.remove(userId));
        emitter.onError(e -> emitters.remove(userId));
        
        // 发送一个初始事件保持连接
        try {
            emitter.send(SseEmitter.event().name("INIT").data("连接已建立"));
        } catch (IOException e) {
            emitter.completeWithError(e);
        }
        
        emitters.put(userId, emitter);
        return emitter;
    }
    
    @PostMapping("/publish/{userId}")
    public ResponseEntity<String> publish(@PathVariable String userId, @RequestBody String message) {
        SseEmitter emitter = emitters.get(userId);
        if (emitter != null) {
            try {
                emitter.send(SseEmitter.event()
                    .name("MESSAGE")
                    .data(message));
                return ResponseEntity.ok("消息已发送");
            } catch (IOException e) {
                emitters.remove(userId);
                return ResponseEntity.internalServerError().body("发送失败");
            }
        } else {
            return ResponseEntity.notFound().build();
        }
    }
    
    @PostMapping("/broadcast")
    public ResponseEntity<String> broadcast(@RequestBody String message) {
        List<String> deadEmitters = new ArrayList<>();
        
        emitters.forEach((userId, emitter) -> {
            try {
                emitter.send(SseEmitter.event()
                    .name("BROADCAST")
                    .data(message));
            } catch (IOException e) {
                deadEmitters.add(userId);
            }
        });
        
        deadEmitters.forEach(emitters::remove);
        return ResponseEntity.ok("广播消息已发送");
    }
}
```

前端实现：

```javascript
function connectSSE() {
    const eventSource = new EventSource('/api/sse/subscribe/user123');
    
    eventSource.addEventListener('INIT', function(event) {
        console.log(event.data);
    });
    
    eventSource.addEventListener('MESSAGE', function(event) {
        console.log('收到消息: ' + event.data);
    });
    
    eventSource.addEventListener('BROADCAST', function(event) {
        console.log('收到广播: ' + event.data);
    });
    
    eventSource.onerror = function() {
        eventSource.close();
        // 可以在这里实现重连逻辑
        setTimeout(connectSSE, 5000);
    };
}
```

优点：

+ 真正的服务器推送，节省资源
+ 自动重连机制
+ 支持事件类型区分
+ 相比WebSocket更轻量

缺点：

+ 单向通信，客户端无法通过SSE向服务器发送数据
+ 连接数限制，浏览器对同一域名的SSE连接数有限制
+ IE浏览器不支持

### 4\. WebSocket

原理：WebSocket是一种双向通信协议，在单个TCP连接上提供全双工通信通道。

SpringBoot配置：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MessageWebSocketHandler(), "/ws/messages")
                .setAllowedOrigins("*");
    }
}

public class MessageWebSocketHandler extends TextWebSocketHandler {
    
    private static final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        String userId = extractUserId(session);
        sessions.put(userId, session);
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        // 处理从客户端接收的消息
        String payload = message.getPayload();
        // 处理逻辑...
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        String userId = extractUserId(session);
        sessions.remove(userId);
    }
    
    private String extractUserId(WebSocketSession session) {
        // 从session中提取用户ID
        return session.getUri().getQuery().replace("userId=", "");
    }
    
    // 发送消息给指定用户
    public static void sendToUser(String userId, String message) {
        WebSocketSession session = sessions.get(userId);
        if (session != null && session.isOpen()) {
            try {
                session.sendMessage(new TextMessage(message));
            } catch (IOException e) {
                sessions.remove(userId);
            }
        }
    }
    
    // 广播消息
    public static void broadcast(String message) {
        sessions.forEach((userId, session) -> {
            if (session.isOpen()) {
                try {
                    session.sendMessage(new TextMessage(message));
                } catch (IOException e) {
                    sessions.remove(userId);
                }
            }
        });
    }
}
```

前端实现：

```javascript
function connectWebSocket() {
    const socket = new WebSocket('ws://localhost:8080/ws/messages?userId=user123');
    
    socket.onopen = function() {
        console.log('WebSocket连接已建立');
        // 可以发送一条消息
        socket.send(JSON.stringify({type: 'JOIN', content: '用户已连接'}));
    };
    
    socket.onmessage = function(event) {
        const message = JSON.parse(event.data);
        console.log('收到消息:', message);
    };
    
    socket.onclose = function() {
        console.log('WebSocket连接已关闭');
        // 可以在这里实现重连逻辑
        setTimeout(connectWebSocket, 5000);
    };
    
    socket.onerror = function(error) {
        console.error('WebSocket错误:', error);
        socket.close();
    };
}
```

优点：

+ 全双工通信，服务器和客户端可以随时相互发送数据
+ 实时性最好，延迟最低
+ 效率高，建立连接后无需HTTP头，数据传输量小
+ 支持二进制数据

缺点：

+ 实现相对复杂
+ 对服务器要求高，需要处理大量并发连接
+ 可能受到防火墙限制

### 5\. STOMP (基于WebSocket)

原理：STOMP (Simple Text Oriented Messaging Protocol) 是一个基于WebSocket的简单消息传递协议，提供了更高级的消息传递模式。

SpringBoot配置：

```less
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // 启用简单的基于内存的消息代理
        registry.enableSimpleBroker("/topic", "/queue");
        // 设置应用的前缀
        registry.setApplicationDestinationPrefixes("/app");
        // 设置用户目的地前缀
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOrigins("*")
                .withSockJS(); // 添加SockJS支持
    }
}

@Controller
public class MessageController {
    
    private final SimpMessagingTemplate messagingTemplate;
    
    public MessageController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }
    
    // 处理客户端发送到/app/sendMessage的消息
    @MessageMapping("/sendMessage")
    public void processMessage(String message) {
        // 处理消息...
    }
    
    // 处理客户端发送到/app/chat/{roomId}的消息，并广播到相应的聊天室
    @MessageMapping("/chat/{roomId}")
    @SendTo("/topic/chat/{roomId}")
    public ChatMessage chat(@DestinationVariable String roomId, ChatMessage message) {
        // 处理聊天消息...
        return message;
    }
    
    // 发送私人消息
    @MessageMapping("/private-message")
    public void privateMessage(PrivateMessage message) {
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(),  // 接收者的用户名
            "/queue/messages",      // 目的地
            message                 // 消息内容
        );
    }
    
    // REST API发送广播消息
    @PostMapping("/api/broadcast")
    public ResponseEntity<String> broadcast(@RequestBody String message) {
        messagingTemplate.convertAndSend("/topic/broadcast", message);
        return ResponseEntity.ok("消息已广播");
    }
    
    // REST API发送私人消息
    @PostMapping("/api/private-message/{userId}")
    public ResponseEntity<String> sendPrivateMessage(
            @PathVariable String userId,
            @RequestBody String message) {
        messagingTemplate.convertAndSendToUser(userId, "/queue/messages", message);
        return ResponseEntity.ok("私人消息已发送");
    }
}
```

前端实现：

```javascript
const stompClient = new StompJs.Client({
    brokerURL: 'ws://localhost:8080/ws',
    connectHeaders: {
        login: 'user',
        passcode: 'password'
    },
    debug: function (str) {
        console.log(str);
    },
    reconnectDelay: 5000,
    heartbeatIncoming: 4000,
    heartbeatOutgoing: 4000
});

stompClient.onConnect = function (frame) {
    console.log('Connected: ' + frame);
    
    // 订阅广播消息
    stompClient.subscribe('/topic/broadcast', function (message) {
        console.log('收到广播: ' + message.body);
    });
    
    // 订阅特定聊天室
    stompClient.subscribe('/topic/chat/room1', function (message) {
        const chatMessage = JSON.parse(message.body);
        console.log('聊天消息: ' + chatMessage.content);
    });
    
    // 订阅私人消息
    stompClient.subscribe('/user/queue/messages', function (message) {
        console.log('收到私人消息: ' + message.body);
    });
    
    // 发送消息到聊天室
    stompClient.publish({
        destination: '/app/chat/room1',
        body: JSON.stringify({
            sender: 'user123',
            content: '大家好！',
            timestamp: new Date()
        })
    });
    
    // 发送私人消息
    stompClient.publish({
        destination: '/app/private-message',
        body: JSON.stringify({
            sender: 'user123',
            recipient: 'user456',
            content: '你好，这是一条私信',
            timestamp: new Date()
        })
    });
};

stompClient.onStompError = function (frame) {
    console.error('STOMP错误: ' + frame.headers['message']);
    console.error('Additional details: ' + frame.body);
};

stompClient.activate();
```

优点：

+ 高级消息模式：主题订阅、点对点消息传递
+ 内置消息代理，简化消息路由
+ 支持消息确认和事务
+ 框架支持完善，SpringBoot集成度高
+ 支持认证和授权

缺点：

+ 学习曲线较陡
+ 资源消耗较高
+ 配置相对复杂

## 三、方案对比与选择建议

| 方案 | 实时性 | 双向通信 | 资源消耗 | 实现复杂度 | 浏览器兼容性 |
| --- | --- | --- | --- | --- | --- |
| 短轮询 | 低 | 否 | 高 | 低 | 极好 |
| 长轮询 | 中 | 否 | 中 | 中 | 好 |
| SSE | 高 | 否(单向) | 低 | 低 | IE不支持 |
| WebSocket | 极高 | 是 | 低 | 高 | 良好(需考虑兼容) |
| STOMP | 极高 | 是 | 中 | 高 | 良好(需考虑兼容) |

选择建议：

1. 简单通知场景：对实时性要求不高，可以选择短轮询或长轮询
2. 服务器单向推送数据：如实时数据展示、通知提醒等，推荐使用SSE
3. 实时性要求高且需双向通信：如聊天应用、在线游戏等，应选择WebSocket
4. 复杂消息传递需求：如需要主题订阅、点对点消息、消息确认等功能，推荐使用STOMP
5. 需要考虑老旧浏览器：应避免使用SSE和WebSocket，或提供降级方案

## 四、总结

在SpringBoot中实现网页消息推送，有多种技术方案可选，每种方案都有其适用场景：

+ 短轮询：最简单但效率最低，适合非实时性要求的场景
+ 长轮询：改进版的轮询，降低了服务器负载，提高了实时性
+ SSE：轻量级的服务器推送技术，适合单向通信场景
+ WebSocket：功能最强大的双向通信方案，适合高实时性要求场景
+ STOMP：基于WebSocket的消息协议，提供了更高级的消息传递功能

选择合适的推送技术需要根据业务需求、性能要求和浏览器兼容性等因素综合考虑。在实际应用中，也可以结合多种技术，提供优雅降级方案，确保在各种环境下都能提供良好的用户体验。

