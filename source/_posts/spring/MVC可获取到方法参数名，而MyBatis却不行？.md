---
title: 【转载】为何springMVC可获取到方法参数名，而MyBatis却不行？
tags: spring
categories: spring
abbrlink: 2790402318
date: 2021-05-11 16:50:00
---

> 原文链接：[为何springMVC可获取到方法参数名，而MyBatis却不行？](https://cloud.tencent.com/developer/article/1497751)

`Spring MVC`和`MyBatis`作为当下最为流行的两个框架，大家平时开发中都在用。如果你往深了一步去思考，你应该会有这样的疑问：

- 在使用`Spring MVC`的时候，即使不使用注解，只要参数名和请求参数的key对应上了，就能自动完成数值的封装
- 在使用`MyBatis`（接口模式）时，接口方法向xml里的SQL语句传参时，必须使用`@Param('')`指定key值，在SQL中才可以取到。

**为什么`Spring MVC`可以动态取到方法参数名称，而`MyBatis`的`Mapper`接口却无法支持**？

<!--more-->

## 问题发现

大家都知道，`.java`文件必须经过`javac`编译成`.class`文件才能被JVM执行。**而在编译的时候，默认是不会保留方法参数名称的**，取而代之的是`arg0`、`arg1`等表示。因此，**想在运行时通过`.class`字节码直接拿到方法的参数名称是不可能做到的**。

如下示例，很明显就是获取不到参数名称：

```java
public static void main(String[] args) throws NoSuchMethodException {
    Method method = Main.class.getMethod("test1", String.class, Integer.class);
    int parameterCount = method.getParameterCount();
    Parameter[] parameters = method.getParameters();

    // 打印输出：
    System.out.println("方法参数总数：" + parameterCount);
    Arrays.stream(parameters).forEach(p -> System.out.println(p.getType() + "----" + p.getName()));
}
```

打印内容为：

```text
方法参数总数：2
class java.lang.String----arg0
class java.lang.Integer----arg1
```


