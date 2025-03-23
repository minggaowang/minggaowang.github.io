## MySQL中like的模糊查询如何优化

当然还可以ES等 这里只说mysql怎么搞

### 典型回答

在MySQL中，使用like进行模糊查询，在一定情况下是无法使用索引的。如下所示：

●当like值前后都有匹配符时%abc%，无法使用索引

●当like值前有匹配符时%abc，无法使用索引

●当like值后有匹配符时'abc%'，可以使用索引

那么，like %abc真的无法优化了吗？

我们之所以会使用%abc来查询说明表中的name可能包含以abc结尾的字符串，如果以abc%说明有以abc开头的字符串。

假设我们要向表中的name写入123abc，我们可以将这一列反转过来，即cba321插入到一个冗余列v\_name中，并为这一列建立索引：

接下来在查询的时候，我们就可以使用v\_name列进行模糊查询了

当然这样看起来有点麻烦，表中如果已经有了很多数据，还需要利用update语句反转name到v\_name中，如果数据量大了（几百万或上千万条记录）更新一下v\_name耗时也比较长，同时也会增大表空间。

### MySQL5.7.6之后，新增了虚拟列功能

幸运的是在MySQL5.7.6之后，新增了虚拟列功能（如果不是>=5.7.6，只能用上面的土方法）为一个列建立一个虚拟列，并为虚拟列建立索引，在查询时where中like条件改为虚拟列，就可以使用索引了。

我们再进行查询，就会走索引了

当然如果你要查询like 'abc%'和like '%abc'，你只需要使用一个union

可以看到，除了union result合并俩个语句，另外俩个查询都已经走索引了。如果你只想需要查询name，甚至可以使用覆盖索引进一步提升性能

虚拟列可以指定为VIRTUAL或STORED，VIRTUAL不会将虚拟列存储到磁盘中，在使用时MySQL会现计算虚拟列的值，STORED会存储到磁盘中，相当于我们手动创建的冗余列。所以：如果你的磁盘足够大，可以使用STORED方式，这样在查询时速度会更快一些。

如果你的数据量级较大，不使用反向查询的方式耗时会非常高。你可以使用如下sql测试虚拟列的效果：

```java
/* 建表 */

CREATE TABLE test (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  INDEX idx_name (name)
) CHARACTER SET utf8;


/* 创建一个存储过程，向test表中写入2000000条数据，200条数据中abc字符前包含一些随机字符（用于测试like '%abc'的情况），200条数据中abc字符后包含一些随机字符（用于测试like 'abc%'的情况），其余行不包含abc字符 */

DELIMITER //

CREATE PROCEDURE InsertTestData()
BEGIN
  DECLARE i INT DEFAULT 1;
  
  WHILE i <= 2000000 DO
    IF i <= 200 THEN
      SET @randomPrefix1 = CONCAT(CHAR(FLOOR(RAND() * 26) 65), CHAR(FLOOR(RAND() * 26) 97), CHAR(FLOOR(RAND() * 26) 48));
      SET @randomString1 = CONCAT(CHAR(FLOOR(RAND() * 26) 65), CHAR(FLOOR(RAND() * 26) 97), CHAR(FLOOR(RAND() * 26) 48));
      SET @randomName1 = CONCAT(@randomPrefix1, @randomString1, 'abc');
      INSERT INTO test (name) VALUES (@randomName1);
    ELSEIF i <= 400 THEN
      SET @randomString2 = CONCAT(CHAR(FLOOR(RAND() * 26) 65), CHAR(FLOOR(RAND() * 26) 97), CHAR(FLOOR(RAND() * 26) 48));
      SET @randomName2 = CONCAT('abc', @randomString2);
      INSERT INTO test (name) VALUES (@randomName2);
    ELSE
      SET @randomName3 = CONCAT(CHAR(FLOOR(RAND() * 26) 65), CHAR(FLOOR(RAND() * 26) 97), CHAR(FLOOR(RAND() * 26) 48));
      INSERT INTO test (name) VALUES (@randomName3);
    END IF;
    
    SET i = i 1;
  END WHILE;
END //

DELIMITER ;

/* 调用存储过程，这里执行的会很慢 */

call InsertTestData();

/* 建立虚拟列 */
alter table test add column `v_name` varchar(50) generated always as (reverse(name));
/* 为虚拟列创建索引 */
alter table test add index `idx_name_virt`(v_name);


/* 使用虚拟列模糊查询 */
select * from test where v_name like 'cba%'
union
select * from test where name like 'abc%'



/* 不使用虚拟列模糊查询 */
select * from test where name like 'abc%'
union
select * from test where name like '%abc'
```

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/34326a3d817b4f108db9ab0bc0b28332~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCO56uv56iL5bqP5ZGYQXNrYQ==:q75.awebp?rk3s=f64ab15b&x-expires=1743219370&x-signature=fCgVyZDbg5UPhMgSeAg4z4h6SJk%3D)

![image.png](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/b33e7366927d443a81c67a8c50d38312~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg5ZCO56uv56iL5bqP5ZGYQXNrYQ==:q75.awebp?rk3s=f64ab15b&x-expires=1743219370&x-signature=srZZoguhjN0uZEFcBg1HOGxVjM0%3D)

## MySQL5.7.6 虚拟列功能

MySQL 5.7 引入了虚拟列（Generated Columns），这些列的值是通过表达式计算得出的，而不是直接存储在表中的。虚拟列可以分为两种类型：VIRTUAL 和 STORED。以下是虚拟列的好处和坏处：

### 好处

1. 简化查询：
    
   虚拟列可以简化复杂的查询，尤其是当查询中需要频繁使用某个表达式时。通过将表达式定义为虚拟列，可以直接查询该列，而不需要在每次查询时重复计算。
2. 数据一致性：
    
   虚拟列的值是根据其他列的值自动计算的，因此可以确保数据的一致性。如果基础列的值发生变化，虚拟列的值会自动更新，避免了手动维护数据一致性的麻烦。
3. 减少冗余：
    
   使用虚拟列可以避免存储冗余数据。例如，如果你需要根据某些列的值计算出一个结果，并且这个结果不需要频繁更新，可以使用虚拟列来动态计算，而不需要将结果存储在表中。
4. 索引支持：
    
   虚拟列可以被索引，这可以显著提高查询性能。特别是当虚拟列的计算结果经常用于查询条件时，创建索引可以加速查询。
5. 灵活性：
    
    虚拟列可以根据需要定义复杂的表达式，提供更高的灵活性。你可以根据业务需求动态生成数据，而不需要修改表结构或应用程序代码。

### 坏处

1. 性能开销：
    
    VIRTUAL 虚拟列的值在每次查询时动态计算，这可能会增加查询的计算开销，尤其是在表达式复杂或数据量大的情况下。虽然 STORED 虚拟列的值是预先计算并存储的，但在插入或更新数据时会有额外的计算和存储开销。
2. 存储空间：
    
    STORED 虚拟列的值是实际存储在表中的，因此会增加表的存储空间。如果虚拟列的计算结果较大或表中有大量数据，这可能会导致存储需求显著增加。
3. 复杂性增加：
    
    虚拟列的定义可能会增加表结构的复杂性，尤其是在定义复杂的表达式时。这可能会使表的设计和维护变得更加困难。
4. 兼容性问题：
    
    虚拟列是 MySQL 5.7 引入的特性，因此在较旧的 MySQL 版本中无法使用。如果你的应用程序需要兼容旧版本的 MySQL，使用虚拟列可能会导致兼容性问题。
5. 索引限制：
    
    虽然虚拟列可以被索引，但并不是所有的表达式都支持索引。某些复杂的表达式可能无法创建索引，这可能会限制虚拟列在查询优化中的应用。

### 总结

虚拟列在 MySQL 5.7 中提供了强大的功能，可以简化查询、提高数据一致性并减少冗余。然而，它们也可能带来性能开销、存储空间增加和复杂性提升等问题。在使用虚拟列时，需要根据具体的业务需求和性能要求进行权衡，确保其带来的好处大于潜在的缺点。

