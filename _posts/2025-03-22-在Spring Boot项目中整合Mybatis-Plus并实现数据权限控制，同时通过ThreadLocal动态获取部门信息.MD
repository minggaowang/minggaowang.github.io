# Spring Boot项目中整合Mybatis-Plus并实现数据权限控制，同时通过`ThreadLocal`动态获取部门

---

### 1. 添加依赖

在`pom.xml`中添加Mybatis-Plus和Spring Boot的相关依赖。

```xml
<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- MyBatis-Plus -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>最新版本号</version>
    </dependency>
    <!-- MySQL Connector -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

---

### 2. 配置数据源

在`application.yml`中配置数据库连接信息。

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/your_database?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC
    username: your_username
    password: your_password
    driver-class-name: com.mysql.cj.jdbc.Driver
```

---

### 3. 创建实体类

创建与数据库表对应的实体类，并使用Mybatis-Plus的注解。

```java
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;

@TableName("user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Long departmentId; // 假设部门ID用于数据权限控制

    // getters and setters
}
```

---

### 4. 创建Mapper接口

创建Mapper接口，并继承`BaseMapper`。

```java
import com.baomidou.mybatisplus.core.mapper.BaseMapper;

public interface UserMapper extends BaseMapper<User> {
}
```

---

### 5. 实现数据权限控制

通过Mybatis-Plus的拦截器实现数据权限控制，并结合`ThreadLocal`动态获取当前用户的部门信息。

#### 5.1 定义`ThreadLocal`工具类

用于存储当前用户的部门信息。

```java
public class UserContextHolder {
    private static final ThreadLocal<Long> DEPT_ID_HOLDER = new ThreadLocal<>();

    public static void setDeptId(Long deptId) {
        DEPT_ID_HOLDER.set(deptId);
    }

    public static Long getDeptId() {
        return DEPT_ID_HOLDER.get();
    }

    public static void clear() {
        DEPT_ID_HOLDER.remove();
    }
}
```

#### 5.2 创建拦截器

在拦截器中从`ThreadLocal`获取部门信息，并动态拼接SQL条件。

```java
import com.baomidou.mybatisplus.core.toolkit.PluginUtils;
import com.baomidou.mybatisplus.extension.plugins.inner.InnerInterceptor;
import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.expression.LongValue;
import net.sf.jsqlparser.expression.operators.relational.EqualsTo;
import net.sf.jsqlparser.schema.Column;
import net.sf.jsqlparser.statement.select.PlainSelect;
import net.sf.jsqlparser.statement.select.Select;
import net.sf.jsqlparser.statement.select.SelectBody;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.SystemMetaObject;

import java.sql.Connection;
import java.util.Properties;

@Intercepts({@Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})})
public class DataScopeInterceptor implements InnerInterceptor {

    @Override
    public void beforePrepare(StatementHandler sh, Connection connection, Integer transactionTimeout) {
        MetaObject metaObject = SystemMetaObject.forObject(sh);
        BoundSql boundSql = (BoundSql) metaObject.getValue("delegate.boundSql");
        String sql = boundSql.getSql();
        Select selectStatement = (Select) CCJSqlParserUtil.parse(sql);
        SelectBody selectBody = selectStatement.getSelectBody();
        if (selectBody instanceof PlainSelect) {
            PlainSelect plainSelect = (PlainSelect) selectBody;
            // 从ThreadLocal获取当前用户的部门ID
            Long departmentId = UserContextHolder.getDeptId();
            if (departmentId != null) {
                EqualsTo equalsTo = new EqualsTo();
                equalsTo.setLeftExpression(new Column("department_id"));
                equalsTo.setRightExpression(new LongValue(departmentId));
                if (plainSelect.getWhere() == null) {
                    plainSelect.setWhere(equalsTo);
                } else {
                    plainSelect.getWhere().addExpressions(equalsTo);
                }
            }
        }
        metaObject.setValue("delegate.boundSql.sql", selectStatement.toString());
    }
}
```

#### 5.3 注册拦截器

在Spring Boot配置类中注册自定义的拦截器。

```java
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new DataScopeInterceptor());
        return interceptor;
    }
}
```

---

### 6. 使用ThreadLocal设置部门信息

在Controller或Service中设置当前用户的部门信息。

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public List<User> getUsers() {
        // 假设当前用户的部门ID为1（实际应从登录信息中获取）
        UserContextHolder.setDeptId(1L);
        List<User> users = userService.getUsersByDepartment();
        UserContextHolder.clear(); // 清除ThreadLocal数据
        return users;
    }
}
```

---

### 7. Service层调用Mapper

在Service层中注入Mapper接口并使用。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public List<User> getUsersByDepartment() {
        return userMapper.selectList(null); // 查询时会自动添加数据权限过滤条件
    }
}
```

---

### 8. 总结

通过以上步骤，你可以实现以下功能：

1. 使用`ThreadLocal`动态存储当前用户的部门信息。
2. 通过Mybatis-Plus拦截器动态拼接SQL，实现数据权限控制。
3. 在多用户场景下，根据当前用户的部门信息过滤数据。

这种方式灵活且易于扩展，适合复杂的权限控制场景。

