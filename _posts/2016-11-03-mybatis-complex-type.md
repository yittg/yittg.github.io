---
layout: post
title: MyBatis 复杂类型
tags: [java, mybatis]
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

如果现在我想返回一个关于 Mty 的 `Map<String, Mty>` 的结构（例如以 `yg` 字段作为 Key）可以怎样呢，事实上 MyBatis 的内置类型中包括 `Map/Hashmap`，并且可以直接作为返回类型写在 Mapper 中，让我们试试

```java
Map<String, Mty> getMty(String f);
```
```xml
<select id="getMty" resultType="hashmap" >
    SELECT * FROM mty WHERE yf = #{f};
</select>
```
然后发现可以正常执行（在只有一行数据匹配时），但是结果是错的，至少不是想要的那样，讲真这里完全没告诉它用什么作为 Key，也是难为它了。错误的结果返回的是 column\_name 到 column\_content 的一个 Map，所以本质上类型也是错误的，但这也可能是一种需求：返回名称到值的映射（正确类型应该是 `Map<String, Object>`）。

> !! • MyBatis 在返回像 List，Map 这种泛型结果填充数据时不做或者说做不到强制类型检查（得益于泛型机制，哈哈），将数据转换后直接塞进名称相同的字段中，然后它做到了 :) 。

那么有没有办法做到上面说的这种需求呢，事实上在返回 Map 结构时，MyBatis 提供了指定 Key 的方法，当然就是另一种效果了。注意到 `resultType` 是 Value 的类型（就像它在返回 List 结构时一样，是指定内部的类型）。

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
上面说到的方法是通过 `resultMap` 将不同表中的字段组合成一个复杂的结构，那么如果我想将 `Mta` 中的 `Mty ay` 字段存为 `mta` 中的一列呢（以字符串的形式）。想象一下，像 `Integer` `String` 这样的基本类型， MyBatis 是怎么准确匹配和处理的呢？这源于它内置实现的 `jdbcType`，`javaType` 关联的 `TypeHandler`。而这个是可以自定义的，也就是说你可以让 MyBatis  按照你的意思将一个类型存成什么样。 

还是从最初开始，像下面这样写入一条记录。

```java
Mta {
   String af;
   String ak;
   Mty ay;
}

int addMta(Mta a);
```
```xml
<insert id="addMta" parameterType="Mta" >
    INSERT INTO `mta`
    (`af`, `ak`, `ay`)
    VALUES (#{af}, #{ak}, #{ay})
</insert>
```

你会发现 MyBatis 会告诉你 `ay` 这个参数的 `TypeHandler` 是 `null`，所以需要定制一个属于它的 handler，然后通过 `MappedTypes` 告诉 MyBatis 这是针对什么类型的 hanler。

```java
@MappedTypes(value = Mty.class)
public class MtyTypeHandler implements TypeHandler<Mty> {
...
}
```
这里 MyBatis 提供了一共更方便的抽象类 `BaseTypeHandler`， 这个类通过继承 `TypeReference` 来达到绑定类型的目的（注册 handler 的时候，MyBatis 会去尝试访问这个类型）。

好的， 继续。这次往里面放一个 Map 结构也没有什么问题了吧，像之前声明的那样。类似的，定义个 
`Map<String, Mty>` 类型的 handler。

```java
public class MtyTypeHandler extends BaseTypeHandler<Map<String, Mty>> {
...
}
```
一如预期的， 它工作的很好 :) ，而且这次结果也是对的，嗯！但是，你会发现它实际绑定的是 `Map` 这个类型（而不是 `Map<String, Mty>`，又是这样），就是说所有的 `Map` 类型的参数都会匹配上这个 handler，这显然不是我们想要的。

这里可以有一个做法，就是再次通过 `MappedTypes` 绑定一个类型，这个类型的作用就是让所有不明确指向它时都不会使用它，然后在使用场景通过 `typeHandler` 参数明确指向这个类型来处理，这样相对便宜。或者另一个做法放弃 `BaseTypeHandler` 的福利，同时也就不通过 `TypeReference` 注册类型。

```java
@MappedTypes(value = MtyTypeHandler.class)
public class MtyTypeHandler extends BaseTypeHandler<Map<String, Mty>>
```

```xml
resultMap:
<result property="ay" column="ay" typeHandler="...MtyTypeHandler" />

parameter:
#{ay, typeHandler=...MtyTypeHandler}
```

### 就这样吧

总之， MyBatis 在数据类型处理方面还是比较灵活的，通常情况下很简单的使用就能满足需求，并且也能支持到足够复杂的场景中。