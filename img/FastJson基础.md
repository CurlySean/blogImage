---
title: FastJson基础
tags: FastJson
categories: Java
description:
---
<h1 id="lXrTl">FastJson基础</h1>
<h2 id="hQtoG">FastJson 简介</h2>
<font style="color:rgb(80, 80, 92);">Fastjson 是 Alibaba 开发的 Java 语言编写的高性能 JSON 库，用于将数据在 JSON 和 Java Object 之间互相转换。</font>

<font style="color:rgb(80, 80, 92);">提供两个主要接口来分别实现序列化和反序列化操作。</font>

`JSON.toJSONString`<font style="color:rgb(80, 80, 92);"> </font><font style="color:rgb(80, 80, 92);">将 Java 对象转换为 json 对象，序列化的过程。</font>

`JSON.parseObject/JSON.parse`<font style="color:rgb(80, 80, 92);"> </font><font style="color:rgb(80, 80, 92);">将 json 对象重新变回 Java 对象；反序列化的过程</font>

+ 所以可以简单的把 json 理解成是一个字符串

<h2 id="af4tf">环境配置</h2>

+ JDK7u21
+ Fastjson 1.2.24

```java
<dependency>  
 <groupId>com.alibaba</groupId>  
 <artifactId>fastjson</artifactId>  
 <version>1.2.24</version>  
</dependency>
```

<h2 id="JGHVL">简单小demo</h2>
定义一个`Person`类，为其设置setter/getter方法

```java
package com;

public class Person {
    private String name;
    private int age;

    public Person() {
        System.out.println("Person");
    }

    public int getAge() {
        System.out.println("getAge");
        return age;
    }

    public void setAge(int age) {
        System.out.println("setAge");
        this.age = age;
    }

    public String getName() {
        System.out.println("getName");
        return name;
    }

    public void setName(String name) {
        System.out.println("setName");
        this.name = name;
    }
}
```

<h3 id="aQ7yy">序列化</h3>
写一个序列化的代码，调用`JSON.toJsonStirng()`来序列化`Person`对象

```java
public class JSONUser {
    public static void main(String[] args) throws Exception {
        String s = "{\"@type\":\"com.Person\",\"age\":18,\"name\":\"abc\"}";

        Person person = new Person();
        person.setAge(18);
        String jsonString = JSON.toJSONString(person);
        System.out.println(jsonString);
    }
}
```

![](https://cdn.jsdelivr.net/gh/CurlySean/blogImage@main/img/1741581603504-58ed2f84-07a8-4a8c-9c0d-4401d51a7592.png)

<h3 id="mvgXU">反序列化</h3>
写一个反序列化的代码，调用`JSONObject.parseObject()`来反序列化`Person`对象

（当需要还原出private的属性时，需要在JSON.parseObject/JSON.parse中加上Feature.SupportNonPublicField参，当然一般没人给私有属性加setter）

```java
public class JSONUser {
    public static void main(String[] args) throws Exception {
        String s = "{\"@type\":\"com.Person\",\"age\":18,\"name\":\"abc\"}";

        Object parse = JSON.parse(s);
        System.out.println(parse);
    }
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1741582884850-efa7c14e-964b-4ed5-80b9-b62066959094.png)

<h4 id="cVjXD">小知识</h4>
Fastjson反序列化采用两个反序列化方法，分别为

+ `<font style="color:rgb(83, 83, 96);background-color:rgb(242, 242, 242);">JSON.parseObject()</font>`
+ `<font style="color:rgb(83, 83, 96);background-color:rgb(242, 242, 242);">JSON.parse()</font>`

`parseObject()`：返回`fastjsonJSONObject`类

`parse()`：返回我们的类

下面我们可以看到，`parseObject()`返回<font style="color:rgb(80, 80, 92);">parseObject类，而</font>`<font style="color:rgb(80, 80, 92);">parse()</font>`<font style="color:rgb(80, 80, 92);">返回我们的User类</font>

![](https://cdn.jsdelivr.net/gh/CurlySean/blogImage@main/img/1740451295154-af89d742-b770-477e-a8d1-62c93c168bab.png)

但是可以通过在parseObject参数中传入类，达到和parse相同效果（也可以传入Student.class）

```java
parseObject(input,Object.class)
```

![](https://cdn.jsdelivr.net/gh/CurlySean/blogImage@main/img/1740451464436-ea6fe816-0714-4966-bb29-c00b3f4cc149.png)

<h2 id="l8PMW">Fastjson反序列化漏洞</h2>
fastjson在反序列化字符串时，会寻找`@type`中的类，在反序列化过程中会自动调用该类的setter和getter方法，但并不是所有getter和setter都会被调用

以下是满足条件的setter和getter的条件（可以根据源码分析出来，这里不多说了）：

**满足条件的setter**

+ 非静态函数
+ 返回类型为void或当前类
+ 参数个数为1个

**满足条件的getter**

+ 非静态方法
+ 无参数
+ **返回值类型继承自Collection或Map或AtomicBoolean或AtomicInteger或AtomicLong**

<h3 id="Z1BNv">漏洞原理</h3>
Fastjson拥有自己的一套实现序列化和反序列化的机制，针对不同版本的Fastjson反序列化漏洞，原理都是一样的，只是针对不同黑名单的绕过利用

攻击者传入一个恶意构造的JSON字符串，Fastjson在反序列化字符串时，得到恶意类并执行恶意类的恶意函数，导致恶意代码执行



我们看之前的代码Demo，他会调用该类的 构造方法、getter、setter方法，若这些方法中存在危险方法的话，即存在Fastjson的 反序列化漏洞

```java
String s = "{\"@type\":\"com.Person\",\"age\":18,\"name\":\"abc\"}";
Object parse = JSON.parse(s);
```

![](https://cdn.jsdelivr.net/gh/CurlySean/blogImage@main/img/1741583089073-ac0818a6-f11f-4dfb-8a08-5a00b0f4b02a.png)

<h3 id="mwThh">POC写法</h3>
一般Fastjson反序列化的POC写法如下

```java
{
"@type":"xxx.xxx.xxx",
"xxx":"xxx",
...
}
```

<h2 id="olZbj">小结</h2>
在学习过程中，发现fastjson的好多东西都没学到，回来重新学习一下，前两天有点忙，所以博客没来得及更新QAQ