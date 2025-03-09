在 Spring Boot 中解决跨域问题（CORS）有以下几种常用方式，根据不同的场景选择合适的方法：

---

### **1. 使用 `@CrossOrigin` 注解（局部配置）**

直接在 **Controller 类或方法** 上添加注解，允许指定来源的跨域请求。

```java
@RestController
public class MyController {

    @CrossOrigin(origins = "http://localhost:3000") // 允许特定来源
    @GetMapping("/hello")
    public String hello() {
        return "Hello World";
    }
}
```

#### **特点**：

- 适合单个接口或控制器。
- 简单直接，但需要为每个需要跨域的接口重复配置。

---

### **2. 全局配置（推荐）**

通过实现 `WebMvcConfigurer` 接口，配置全局跨域规则。

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**") // 所有接口
                .allowedOrigins("*") // 允许所有来源（生产环境建议指定具体域名）
                .allowedMethods("GET", "POST", "PUT", "DELETE") // 允许的HTTP方法
                .allowedHeaders("*") // 允许所有请求头
                .allowCredentials(false) // 是否允许携带Cookie（设为true时，allowedOrigins不能为*）
                .maxAge(3600); // 预检请求缓存时间（秒）
    }
}
```

#### **特点**：

- **全局生效**，无需为每个接口单独配置。
- 适合统一管理跨域规则。

---

### **3. 使用 `CorsFilter`（更底层控制）**

通过自定义过滤器实现跨域配置，适用于需要更灵活的场景。

```java
@Configuration
public class CorsFilterConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(false); // 是否允许携带Cookie
        config.addAllowedOrigin("*");      // 允许所有来源
        config.addAllowedHeader("*");      // 允许所有请求头
        config.addAllowedMethod("*");      // 允许所有HTTP方法

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config); // 所有路径

        return new CorsFilter(source);
    }
}
```

---

### **4. 结合 Spring Security 配置跨域**

如果项目使用了 Spring Security，需在安全配置中显式启用跨域：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .cors() // 启用跨域
            .and()
            .csrf().disable() // 如果不需要CSRF防护，可以禁用
            .authorizeRequests()
            .anyRequest().permitAll(); // 按需配置权限
    }

    // 可选：自定义Cors配置源
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList("http://localhost:3000"));
        config.setAllowedMethods(Arrays.asList("GET", "POST"));
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

### **5. 配置文件方式（仅限 WebFlux 项目）**

如果是 **Spring WebFlux** 项目（响应式编程），可以通过 `application.yml` 配置：

```yaml
spring:
  webflux:
    cors:
      allowed-origins: "http://localhost:3000"
      allowed-methods: "GET,POST"
      allowed-headers: "*"
```

---

### **常见问题排查**

1. **`allowCredentials(true)` 时，`allowedOrigins` 不能为 `*`**
   需指定具体域名，例如：`.allowedOrigins("http://localhost:3000")`。
2. **Spring Security 中跨域不生效**
   确保在 `HttpSecurity` 配置中调用了 `.cors()`，并检查是否配置了 `CorsConfigurationSource` Bean。
3. **预检请求（OPTIONS）被拦截**
   确保跨域配置中允许 `OPTIONS` 方法，或在 Spring Security 中放行 OPTIONS 请求。

---

### **总结**

- **简单场景**：使用 `@CrossOrigin` 注解。
- **统一管理**：全局配置（`WebMvcConfigurer`）。
- **精细控制**：`CorsFilter` 或 Spring Security 配置。
- **WebFlux 项目**：通过配置文件快速实现。

