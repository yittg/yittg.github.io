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

...
<collection property="ay" ofType="Mty">
    <result property="yf" column="mty_yf" />
    <result property="yg" column="mty_yg" />
</collection>
```

### 主题其实是这样
上面说到的方法是通过 `resultMap` 将不同表中的字段组合成一个复杂的结构，那么如果我想将 `Mta` 中的 `Mty ay` 字段存为 `mta` 中的一列呢（以字符串的形式）。