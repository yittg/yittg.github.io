---
layout: post
title: MyBatis 复杂类型
---

在数据类型方面，[MyBatis](http://www.mybatis.org/mybatis-3/) 提供了基本的 `parameterType` 和 `resultType`，基本能大部分的需求。对于更复杂的需求，MyBatis 提供了 `resultMap`, 对于非基本类型的字段提供了 `typeHandler`。

### 从简单开始

例如有一个复杂类型(简化表示如下)，将他们分别用两个表（`mta`,`mty`）存储，并通过 `*f` 字段关联，那么通过 `resultMap` 很容易获取到一个 `Mta` 的对象。

```java
Mty {
    String yf;
    String yg;
}

Mta {
   String af;
   String ak;
   Mty ay;
}

Mta getMta(String f);
```

```xml
<resultMap id="result" type="Mta">
    <result property="af" column="mta_af" />
    <result property="ak" column="mta_ak" />
    <association property="ay" javaType="Mty">
        <result property="yf" column="mty_yf" />
        <result property="yg" column="mty_yg" />
    </association>
</resultMap>
```

### 如果是这样呢
如果现在 `Mta` 下面是多个 `Mty` 呢，`resultMap` 也很容易做到，只需要将其表示为`collection` 即可。

```java
Mta {
   String af;
   String ak;
   List<Mty> ay;
}

Mta getMta(String f);
```

```xml
...
<collection property="ay" ofType="Mty">
    <result property="yf" column="mty_yf" />
    <result property="yg" column="mty_yg" />
</collection>
```

### 就这样？
不止于此，MyBatis 的 `resultMap` 还有包括 `discriminator`，`autoMapping` 等功能，你甚至可以在一个调用中直接执行两次查询，并且直接映射到对应的结构中，这些文档讲得比较清楚了。

如果现在我想返回一个关于 Mty 的 `Map<String, Mty>` 的结构（例如以 `yf` 字段作为 Key）可以怎样呢，事实上 MyBatis 的内置类型中包括 `Map/Hashmap`，并且可以直接作为返回类型写在 Mapper 中，让我们试试

```java
List<Map<String, Mty>> getMty(String f);
```
```xml
<select id="getMty" resultType="hashmap" >
    SELECT * FROM mty WHERE yf = #{f};
</select>
```
然后可以发现，结果可以正常执行，但是是错的，至少不是想要的那样，讲真这里完全没告诉它用什么作为 Key，也是难为它了。事实上它返回的是 column\_name 到 column\_content 的一个 Map，所以本质上类型其实是错误的，但这也可能是一种需求，返回名称到值的 List （正确类型应该是 `List<Map<String, Object>>`）。

> !! • MyBatis 在返回像 List，Map 这种泛类型结果填充填充数据时不做或者说做不到强制类型检查，将数据转换后直接塞进名称相同的字段中，当然它做到了 :) 。

那么有没有办法做到上面说的这种需求呢，事实上在返回 Map 结构时，MyBatis 提供了指定 Key 的方法，就是另一种效果了。

```java
@MapKey("yg")
Map<String, Mty> getMtys(String f);
```

```xml
<select id="getMtys" resultType="Mty" >
    SELECT * FROM mty WHERE yf = #{f}
</select>
```

回过头来，如果想把这个 Map 结构塞进 `Mta` 结构中，像下面这样，又可以怎么做呢

```java
Mta {
   String af;
   String ak;
   Map<String, Mty> ay;
}
```
好像暂时没有太好的办法了 :( 。

### 继续
上面说到的方法是通过 `resultMap` 将不同表中的字段组合成一个复杂的结构，那么如果我想将 `Mta` 中的 `Mty ay` 字段存为 `mta` 中的一列呢（以字符串的形式）。