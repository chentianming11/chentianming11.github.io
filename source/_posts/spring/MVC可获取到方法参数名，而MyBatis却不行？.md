---
title: 为何springMVC可获取到方法参数名，而MyBatis却不行？
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

但是在使用`SpringMVC`的时候，`Controller`的方法不用注解一样可以完成参数自动映射，例如：

```java
@GetMapping("/test")
public Object test(String name, Integer age) {
    String value = name + "---" + age;
    System.out.println(value);
    return value;
}
```

## ParameterNameDiscoverer

实际上，`Spring`底层是通过`ParameterNameDiscoverer`来实现参数名称获取的，它主要包含3个实现。

- `StandardReflectionParameterNameDiscoverer`：基于`JDK`实现的参数名称发现器，必须在`JDK8`以上版本实现，并且只有在指定`-parameters`才能生效。
- `LocalVariableTableParameterNameDiscoverer`：基于`ASM`实现的参数名称发现器，通过`ASM`提供的通过字节码获取方法的参数名称，支持任何JDK版本。
- `PrioritizedParameterNameDiscoverer`：优先级参数名称发现器，可以认为是具体各个参数名称发现器的聚合管理，按优先级顺序取到一个可用的参数名称解析器进行使用。默认实现为`DefaultParameterNameDiscoverer`。

### StandardReflectionParameterNameDiscoverer

`StandardReflectionParameterNameDiscoverer`实现非常简单，底层直接调用了`JDK`获取参数名称的方法。具体源码如下：

```java
public class StandardReflectionParameterNameDiscoverer implements ParameterNameDiscoverer {

	@Override
	@Nullable
	public String[] getParameterNames(Method method) {
		return getParameterNames(method.getParameters());
	}

	@Override
	@Nullable
	public String[] getParameterNames(Constructor<?> ctor) {
		return getParameterNames(ctor.getParameters());
	}

	// 因为Parameter这个类是JDK8以上才提供的
	@Nullable
	private String[] getParameterNames(Parameter[] parameters) {
		String[] parameterNames = new String[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			Parameter param = parameters[i];
			if (!param.isNamePresent()) {
				return null;
			}
			parameterNames[i] = param.getName();
		}
		return parameterNames;
	}

}
```

需要特别注意的是，`StandardReflectionParameterNameDiscoverer`的使用条件有两个：

- 必须是`JDK8`以上版本。
- 必须编译的时候有带上参数：`javac -parameters`。



### LocalVariableTableParameterNameDiscoverer

`LocalVariableTableParameterNameDiscoverer`是基于`ASM`实现的参数名称发现器，通过`ASM`提供的通过字节码获取方法的参数名称，支持任何JDK版本。

#### 基本使用

```java
public class Main {

    private static final ParameterNameDiscoverer parameterNameDiscoverer = new LocalVariableTableParameterNameDiscoverer();

    public void testArguments(String test, Integer myInteger, boolean booleanTest) {
    }

    public void test() {
    }

    public static void main(String[] args) {
        Method[] methods = Main.class.getDeclaredMethods();
        for (Method method : methods) {
            // 通过ParameterNameDiscoverer拿到参数名称列表
            String[] parameterNames = parameterNameDiscoverer.getParameterNames(method);
            System.out.println("方法：" + method.getName() + " 参数为：" + Arrays.asList(parameterNames));
        }
    }

}
```

输出结果如下：

```text
方法：main 参数为：[args]
方法：test 参数为：[]
方法：testArguments 参数为：[test, myInteger, booleanTest]
```

#### 实现原理

为了便于理解，先简单说说字节码中的两个概念：`LocalVariableTable`和`LineNumberTable`。

##### LineNumberTable

你是否曾经疑问过：`线上程序抛出异常时显示的行号，为啥就恰好就是你源码的那一行呢`？因为JVM执行的是`.class`文件，而该文件的行和`.java`源文件的行肯定是对应不上的，为何行号却能在`.java`文件里对应上？
其实底层就是`LineNumberTable`的作用：**`LineNumberTable`属性存在于代码（字节码）属性中，它建立了字节码偏移量到源代码行号之间的联系**。

##### LocalVariableTable

**`LocalVariableTable`属性建立了方法中的局部变量与源代码中的局部变量之间的对应关系。这个属性也是存在于代码（字节码）中**。
从名字可以看出来：它是局部变量的一个集合，**描述了局部变量和描述符以及和源代码的对应关系**。

下面我使用`javac`和`javap`命令来演示一下这个情况：
`.java`源码如下：

```java
package com.fsx.maintest;
public class MainTest2 {
    public String testArgName(String name,Integer age){
        return null;
    }
}
```

使用`javac MainTest2.java`编译成`.class`字节码，然后使用`javap -verbose MainTest2.class`查看该字节码信息如下：
![LineNumberTable](https://chentianming11.github.io/images/spring/LineNumberTable.png)

从图中可看到，我红色标注出的行号和源码处完全一样，这就解答了我们上面的行号对应的疑问了：`LineNumberTable`它记录着在源代码处的行号。

> Tips：此处并没有，并没有，并没有`LocalVariableTable`。

源码不变，我使用`javac -g MainTest2.java`来编译，再看看对应的字节码信息如下（注意和上面的区别）：
![LocalVariableTable](https://chentianming11.github.io/images/spring/LocalVariableTable.png)

这里多了一个`LocalVariableTable`，即局部变量表，就记录着我们方法入参的形参名字。既然记录着了，这样我们就可以通过分析字节码信息来得到这个名称了。

> javac的调试选项主要包含了三个子选项：`lines`，`source`，`vars`。
如果不使用`-g`来编译,只保留源文件和行号信息；如果使用`-g`来编译那就都有了。

**`ASM`是一个Java字节码操控框架，它能被用来动态生成类或者增强既有类的功能，它能够改变类行为，分析类信息，甚至能够根据用户要求生成新类**。借助于`ASM`，可以很轻松的修改`.class`字节码中的`LocalVariableTable`，从而实现动态获取参数名的功能。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)

