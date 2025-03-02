### 默认参数处理机制

在MyBatis中，当Mapper接口的方法有多个参数且没有使用`@Param`注解时，默认情况下，MyBatis会将这些参数放入一个`Map`中。键名默认为`param1`, `param2`等，或者可以通过索引`0`, `1`来访问。

#### 使用示例

假设有一个Mapper接口方法如下：

```java
public List<User> findUsers(String name, int age);
```

在对应的Mapper XML文件中，可以这样引用参数：

- 使用默认的`param1`, `param2`：
  
  ```xml
  <select id="findUsers" resultType="User">
      SELECT * FROM users WHERE name = #{param1} AND age = #{param2}
  </select>
  ```
- 或者使用索引`0`, `1`：
  
  ```xml
  <select id="findUsers" resultType="User">
      SELECT * FROM users WHERE name = #{0} AND age = #{1}
  </select>
  ```

### 注意事项

1. **可读性和维护性**：虽然这种方式可行，但为了提高代码的可读性和维护性，建议显式地使用`@Param`注解为每个参数指定名称。
   
   示例：
   
   ```java
   public List<User> findUsers(@Param("name") String name, @Param("age") int age);
   ```
   
   对应的Mapper XML文件：
   
   ```xml
   <select id="findUsers" resultType="User">
       SELECT * FROM users WHERE name = #{name} AND age = #{age}
   </select>
   ```
2. **参数顺序敏感**：如果不使用`@Param`注解，参数的顺序非常重要。如果参数顺序发生变化，SQL语句可能会出错。
3. **复杂参数类型**：对于复杂的参数类型（如自定义对象、集合等），不使用`@Param`注解可能会导致难以理解和维护的问题。因此，推荐始终使用`@Param`注解或直接传递POJO对象。
4. **版本差异**：不同版本的MyBatis可能对参数处理机制有不同的实现细节，请确保查阅所使用版本的官方文档以获取最准确的信息。

总结来说，虽然MyBatis提供了默认的参数处理机制，但在实际开发中，为了代码的清晰和易维护性，建议显式使用`@Param`注解。

