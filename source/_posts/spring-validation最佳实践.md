---
title: 还在为参数校验发愁？spring-validation最佳实践指南
tags:
  - spring
categories:
  - spring
abbrlink: 2132811545
date: 2020-04-06 14:41:58
---

最近线上接口受到白帽子攻击，由于后端接口没有严格地进行参数校验，从而导致了系统程序异常和线上脏数据的问题。为了项目中参数校验方式的统一，因此在项目中引入了`spring-validation`。本文主要介绍`spring-validation`在项目中最佳实践方案，希望能帮助大家很快很好的使用`spring-validation`。
<!-- more -->

## 实体命名方式推荐

在Spring项目组中，会存在有很多实体类，良好的命名方式能十分有效的理清项目的整体划分。下面分别介绍`entity`、`DTO`、`VO`实体类命名方式推荐。

### entity

**entity表示实体bean，一般是用于ORM对象关系映射** ，一个实体映射成一张表，一般无业务逻辑代码。负责将数据库中的表记录映射为内存中的Entity对象。

### DTO

**DTO表示数据传输对象（Data Transfer Object） ，用于服务器和客户端之间交互传输使用的**。在spring-web项目中就是用于接收请求参数的对象。

### VO

**数据视图对象**，一般是指返回响应结果的对象。

### 推荐命名

|--dto

&emsp;|--\*DTO.java

|--entity

&emsp;|--\*Entity.java

|--vo

&emsp;|--\*VO.java

## 参数校验实战

spring-validation原则上可以通过注解的形式，用在任何方上执行参数校验。但是推荐**统一在Controller进行参数校验**。下面介绍一下不同场景下的推荐用法。

*spring-boot-web已经集成了参数校验相关依赖，无需另外再引入！*

### 统一异常处理

对于参数校验失败的场景，推荐使用统一异常处理，以下是示例代码：

```java
@ExceptionHandler({MethodArgumentNotValidException.class})
public ResponseEntity handleMethodArgumentNotValidException(MethodArgumentNotValidException ex) {
    BindingResult bindingResult = ex.getBindingResult();
    StringBuilder sb = new StringBuilder("校验失败:");
    for (FieldError fieldError : bindingResult.getFieldErrors()) {
        sb.append(fieldError.getField()).append("：").append(fieldError.getDefaultMessage()).append(", ");
    }
    String msg = sb.toString();
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .body(msg);
}

@ExceptionHandler({ConstraintViolationException.class})
public ResponseEntity handleMethodArgumentNotValidException(ConstraintViolationException ex) {
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .body(ex.getMessage());
}
```

### requestBody参数校验

对于使用requestBody传递的参数，后端使用**DTO对象**进行接收。一般情况下`POST`、`PUT`请求都会采用这种方式。**只要给DTO对象前加上`@Validated`注解实现自动参数校验**。因为同一个DTO类，很可能被多个方法作为参数，**推荐使用校验分组,  @Validated 注解后面指定校验组即可**！

例如有如下DTO类：

```java
Class FooDTO{
    /**
    * 只有在 Adult 分组下，18 岁的限制才会起作用。
    */
    @Min(value = 18,groups = {Adult.class})
    private Integer age;
    /**
    * 接口表示校验组
    */
    public interface Adult{}

    public interface Minor{}
}

```

Controller方法使用spring-validation示例如下：

```java
@PostMapping("/drink")
public String drink(@RequestBody @Validated({Foo.Adult.class}) FooDTO foo) {
    // ...
    return "success";
}
```

DTO对象接收的方式同样也适用于以下场景：

1. x-www-form-urlencoded形式的请求体参数。
2. url查询参数

### requestParam参数校验

对于`GET`请求，后端一般使用**单个字段分别接收**。这种情况下，**Controller类上必须加上@Validated，然后在每个参数前面加上校验注解即可**。例如：@NotEmpty、@Min等。

```java
@RestController
// 一定要加@Validated注解
@Validated
public class TestController {

    @GetMapping(path = "/test")
    public ResponseEntity test(@RequestParam @Email String email, @RequestParam @Size(min = 1, max = 10) String key) {
        // ...
    }
}
```

### 集合校验

如果请求体传递了json数组给后台，希望对数组中的每一项都进行参数校验。此时，**直接使用java.util.Collection下的list或者set来接收数据，参数校验并不会生效**！我们必须使用**自定义list**来接收参数：

```java
public class ValidatorList<E> implements List<E> {

    @Delegate // @Delegate是lombok注解
    @Valid // 一定要加@Valid注解
    public List<E> list = new ArrayList<>();

    // 一定要记得重写toString方法
    @Override
    public String toString() {
        return list.toString();
    }
}
```

Controller方法示例如下：

```java
@PostMapping("/addCountWithEncryptRelationId")
public ResponseEntity addCount(@RequestBody @NotEmpty @Validated(CountDTO.EncryptRelationIdBatch.class) ValidatorList<CountDTO> countDTOList) {
    // ...
}
```

注意：`ValidatorList<CountDTO> countDTOList`参数前面同时加了`@NotEmpty @Validated(CountDTO.EncryptRelationIdBatch.class)`注解，含义如下：

1. `@NotEmpty`表示list不能为空
2. `@Validated(CountDTO.EncryptRelationIdBatch.class)`表示对于countDTOList的每一项，都会使用CountDTO.EncryptRelationIdBatch.class分组的校验逻辑执行校验！

### 嵌套校验

如果DTO字段包含非主数据类型或者字符串，需要加在该字段上加上`@Valid`注解才能执行嵌套数据校验。

```java
public class Item {

    @NotNull(message = "id不能为空")
    @Min(value = 1, message = "id必须为正整数")
    private Long id;

    @NotNull(message = "props不能为空")
    @Size(min = 1, message = "至少要有一个属性")
    @Valid // 嵌套验证必须用@Valid
    private List<Prop> props;
}

public class Prop {

    @NotNull(message = "pid不能为空")
    @Min(value = 1, message = "pid必须为正整数")
    private Long pid;

    @NotNull(message = "vid不能为空")
    @Min(value = 1, message = "vid必须为正整数")
    private Long vid;

    @NotBlank(message = "pidName不能为空")
    private String pidName;

    @NotBlank(message = "vidName不能为空")
    private String vidName;
}
```

注意：**如果Controller层通过@Validated指定了分组，那么嵌套校验会延续使用该分组执行校验！**

### 自定义校验

业务需求总是比框架提供的这些简单校验要复杂的多，我们可以自定义校验来满足我们的需求。自定义 spring validation 非常简单，主要分为两步：

#### 自定义校验注解

假设我们自定义一个加密id的注解。（就是只有由数字或者a-f的字母组成，32-256长度）

```java
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = {EncryptIdValidator.class})
public @interface EncryptId {

    // 默认错误消息
    String message() default "加密id格式错误";

    // 分组
    Class<?>[] groups() default {};

    // 负载
    Class<? extends Payload>[] payload() default {};

    // 指定多个时使用
    @Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE})
    @Retention(RUNTIME)
    @Documented
    @interface List {
        EncryptId[] value();
    }
}
```

我们不需要关注太多东西，使用 spring validation 的原则便是便捷我们的开发，例如 payload，List ，groups，都可以忽略。

#### 编写真正的校验者类

```java
public class EncryptIdValidator implements ConstraintValidator<EncryptId, String> {

    private static final Pattern PATTERN = Pattern.compile("^[a-f\\d]{32,256}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        // 不为null才进行校验
        if (value != null) {
            Matcher matcher = PATTERN.matcher(value);
            return matcher.find();
        }
        return true;
    }
}
```

这样我们就可以使用`@EncryptId`进行参数校验了，是不是很简单！
