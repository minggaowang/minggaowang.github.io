## Mybatis-plus 更新 Null 的策略踩坑记

在一个管理页面，有一个非必填字段被设置成空了并提交更新，再次打开的时候，发现字段还在，并没有被更新成功。

![](https://p3-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/e938c5aecf8d4c4cbe56c5b20c005a30~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgdXpvbmc=:q75.awebp?rk3s=f64ab15b&x-expires=1744707214&x-signature=V3ExyV4E0X3Shl8TcU%2BHB3C2FwM%3D)

使用的数据库映射框架是 Mybatis-plus ，对于Mybatis 在更新字段的时候会对空进行校验，如果字段为 null，会被忽略加入 sql。

## 应对策略

### 推荐策略：使用 `UpdateWrapper` (3.x)

使用 UpdateWrapper 或 LambdaUpdateWrapper

这是官方给出的案例

```less
//updateAllColumnById(entity) // 全部字段更新: 3.0已经移除
mapper.update(
   new User().setName("mp").setAge(3),
   Wrappers.<User>lambdaUpdate()
           .set(User::getEmail, null) //把email设置成null
           .eq(User::getId, 2)
);
//也可以参考下面这种写法
mapper.update(
    null,
    Wrappers.<User>lambdaUpdate()
       .set(User::getAge, 3)
       .set(User::getName, "mp")
       .set(User::getEmail, null) //把email设置成null
       .eq(User::getId, 2)
);
```

通过这种方式，可以达到将字段设置成 null 的效果。

### 其他策略（不推荐并且慎用）

在字段上增加了 `@TableField(updateStrategy = FieldStrategy.IGNORED)`

在 DO 字段上添加这个更新策略，就是不会判断 null，任何时候都加入 sql

**特别注意：这个策略，在高级版本中已经被 deprecated （3.5.3.2）**

![](https://p3-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/a66b867b424d412f9f5004a6e42ec370~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgdXpvbmc=:q75.awebp?rk3s=f64ab15b&x-expires=1744707214&x-signature=Ln8RuvEwNLeKGp4h6bGp7AFfamI%3D)

### 更新丢失风险 FieldStrategy.ALWAYS

意味着在每次更新操作中，都会将实体对象中的所有字段值写入数据库，无论这些字段的值是否为 `null`

风险巨大。

有一张用户表， 在 salary 上添加 FieldStrategy.ALWAYS

(id,name,age,salary)

开发A：通过 id 更新 age， 但是 DO 没有设置 salary了，mybaits-plus 会添加 salary ，最终导致 salary 被更新为 null，导致数据丢失。

这种问题如果不注意，后果十分严重，不建议在 DO 上添加此策略。随着业务发展，人员变动，这种操作会给以后埋下隐患。

另外使用这种策略以后，对于批量更新简直噩梦！慎重！

### 自定义 sql 方式

如果需要更复杂的逻辑来决定何时将字段设置为 `null`，可以选择编写自定义的 XML 映射文件或者使用 `@Select`, `@Update` 等注解来定义 SQL 语句，在其中明确写出 `SET column_name = NULL`

### 其他策略(“曲线救国”)

如果是 String 类型，可以设置成“”，例如： MapStruct 设置一个默认值。

```less
@Mappings({
        @Mapping(source = "name", target = "name", defaultValue = "")
})
UpdateParam convert(Request request);
```

## 结束

1. 谨慎使用策略 FieldStrategy.ALWAYS/ FieldStrategy.IGNORED
2. 更新使用 UpdateWrapper 或 LambdaUpdateWrapper

