---
title: SpringBoot项目整合Swagger自动生成接口文档之最佳实践指南
tags: spring
categories: spring
abbrlink: 1541374189
date: 2020-12-04 19:50:18
---

最近为项目引入`Swagger`来支持自动生成文档功能，发现很多文章仅仅介绍了如何接入以及如何使用的问题。**但是对于实际工程实践，并没有给出相应的最佳实践方案。因此，我重新梳理相关内容以及文档，整理出一套最佳实践指南**。

<!--more-->

## Swagger和Springfox

在正式介绍之前，我们首先要了解`Swagger`和`Springfox`之间的关系。相信在`Spring`项目使用过`Swagger`的同学都知道，在`Spring`项目中是通过`Springfox`来整合`Swagger`功能的。

### Swagger是什么

根据官网的介绍，`Swagger`是一系列用于`Restful API`开发的工具，开源的部分包括：

- OpenAPI Specification：API规范，规定了如何描述一个系统的API
- Swagger Codegen：用于通过API规范生成服务端和客户端代码
- Swagger Editor：用来编写API规范
- Swagger UI：用于展示API规范

**简单来说，我们可以认为`swagger`主要制定并实现了一个`API`规范，用于描述系统的`API`接口**。

### Springfox是什么

简单来说，`Springfox`其实是一个通过扫描并提取代码中的信息，来生成`API`文档的工具，支持`swagger`、`RAML`、`jsonapi`等多种`API`文档的格式。因为我们这里着重讨论`swagger`，因此我们可以简单理解为：**`Springfox`就是整合`SpringMVC`和`swagger`的中间层，以支持自动扫描`Controller`层的接口来生成符合`swagger`API规范的描述数据(`JSON`格式)**。

通过`Springfox`，已经可以将`swagger`与`SpringBoot`项目整合起来了，网上有一大堆整合的文章，这里就不再赘述了，具体可以参考[SpringBoot整合Swagger实战](https://juejin.cn/post/6844903991793418248)。并且官方最近也出了`3.0`版本，具体可以参考：[还在手动整合Swagger？Swagger官方Starter是真的香！](https://juejin.cn/post/6890692970018766856)。

## knife4j简介

虽然直接通过`Springfox`已经可以实现`SpringBoot`接入`Swagger`生成接口文档。但是这种方式有两个弊端：

1. 原生接口文档页面展示不够友好。
2. 使用起来可能比较繁琐。

因此，在这里强烈推荐使用`Knife4j`。`Knife4j`是`Swagger`接口文档服务的通用性解决方案，底层基于`Springfox`实现。最主要的两个特性如下：

1. 重写了前端UI界面，更符合国人使用习惯。
2. 支持快速接入，并提供了很多实用功能增强。
3. 文档友好，直接参考`Knife4j`官网文档就能完成接入。

能让用户以最低成本接入的方式就是最好的方式，而`Knife4j`确实实现的非常好。更多详细信息可以参考官方文档[https://xiaoym.gitee.io/knife4j/documentation/description.html](https://xiaoym.gitee.io/knife4j/documentation/description.html)。

## 最佳实践指南

**实际上，`SpringBoot`的`Controller`层方法已经包含描述该接口详细信息了。当然，因为我们要补充中文说明。因此，最为理想的状况是，接口文档直接能直接根据`Controller`层的方法自动生成，我们顶多为每个参数编写一个中文说明，仅此而已**。

**我觉得完全自描述的接口才是最合理的，同时也是最简便的**。但是，理想很好，现实情况是`Springfox`对此支持的并不算太好。因此，我们需要在实践的时候，找到一种方式来解决这个问题，下面会介绍一些常见问题的解决思路。

### 2.x和3.x版本如何选择

`Knife4j`的`2.x`版本对应`Springfox`的`2.x`版本，其采用的是`Swagger2`规范，相应的`3.x`版本使用的`OpenAPI3`规范。**关于这两个规范，我们不需要了解太多细节，只需要知道`OpenAPI3`规范能够更准确描述接口就行**。而对于`Springfox`而言，`3.x`版本还有一个优势是：**对于`Bean Validation`支持更好，能够从接口自身解析出更详细的接口信息**。因此，**如果条件允许，强烈建议`3.x`版本**！我甚至觉得，如果是`2.x`版本，接入的价值都不大。我始终觉得，在一个本身已经自描述的接口上，再去专门为文档去写各种注释是一件本末倒置的事情。既然已经自描述，那么最佳的解决方案永远都是框架去解析，而不是人工再写一遍！！！

以下是一个`2.x`和`3.x`版本，对同一接口生成的接口文档，相信大家看完会有一个自己的判断！

`2.x`版本：
![swagger2.x](https://chentianming11.github.io/images/spring/swagger2.x.png)

`3.x`版本：
![swagger3.x](https://chentianming11.github.io/images/spring/swagger3.x.png)

**`3.x`版本能解析出更多的接口信息，描述更为准确**。

### 对请求体入参支持不够好

对于`Query`查询参数，`Springfox`的`3.x`版本已经支持的非常好了，我们按之前的方式写接口即可。但是对于请求体参数，现有框架支持的还是不够好。**其中，最为典型的是不支持分组校验**。这样的导致的结果就是，文档上会将整个`DTO`类的字段以及约束全部展示到任何一个使用了该`DTO`接收参数的接口上。很明显，这并不合理，因为分组校验做的就是让该接口指定分组下的约束只对这个接口生效。

针对这个问题，有什么完美的解决方案吗？答案是没有，这里只能做一个取舍。**如果十分看重接口文档准确性的话，只能不使用分组校验，这样的话，意味着每个接口的请求体都要用不同的`DTO`类来接收，以防止互相干扰**。如果不是特别在意，那还是按分组校验来写，实现更简单。目前，我们项目准备选第一种方式。

> 如果不了解参数校验，可以戳这里：[Spring Validation最佳实践及其实现原理，参数校验没那么简单！](https://juejin.cn/post/6856541106626363399)。对于参数校验，强烈建议这种注解式的方式。因为只有注解式，才能真正做到接口自描述。

### 接入和使用示例

#### 引入依赖

```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-spring-boot-starter</artifactId>
    <version>3.0.2</version>
</dependency>
```

#### 编写接口

**只需要额外再为每个字段加中文说明即可**！

```java
@RestController
@RequestMapping("/api/person")
@Validated
@Api(tags = "Person管理")
public class PersonController {


    @ApiOperation("保存用户")
    @PostMapping("savePerson")
    public Result<PersonVO> savePerson(@RequestBody @Valid SavePersonDTO person) {
        PersonVO personVO = new PersonVO().setAge(10).setEmail("xxxxx").setId(1L).setName("哈哈");
        return Result.ok(personVO);
    }

    @ApiOperation("更新用户")
    @PostMapping("updatePerson")
    public Result<PersonVO> updatePerson(@RequestBody @Valid UpdatePersonDTO person) {
        PersonVO personVO = new PersonVO().setAge(10).setEmail("xxxxx").setId(1L).setName("哈哈");
        return Result.ok(personVO);
    }


    @ApiOperation("查询person")
    @GetMapping("queryPerson")
    public Result<List<PersonVO>> queryPerson(
            @ApiParam("用户id") @Min(1000) @Max(10000000) Long id,
            @ApiParam("姓名") @NotNull @Size(min = 2, max = 10) String name,
            @ApiParam("年龄") @NotNull @Max(200) Integer age,
            @ApiParam("邮箱") @Pattern(regexp = "^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+$") String email) {
        List<PersonVO> list = new ArrayList<>();
        list.add(new PersonVO().setAge(10).setEmail("xxxxx").setId(1L).setName("哈哈"));
        return Result.ok(list);
    }

}


@Data
@Accessors(chain = true)
@ApiModel("保存用户DTO类")
public class SavePersonDTO {

    @ApiModelProperty("姓名")
    @NotNull
    @Size(min = 2, max = 10)
    private String name;

    @ApiModelProperty("年龄")
    @NotNull
    @Max(200)
    private Integer age;

    @ApiModelProperty("邮箱")
    @Pattern(regexp = "^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+$")
    private String email;
}

@Data
@Accessors(chain = true)
@ApiModel("更新用户DTO类")
public class UpdatePersonDTO {


    @ApiModelProperty("用户id")
    @NotNull
    private Long id;

    @ApiModelProperty("姓名")
    @Size(min = 2, max = 10)
    private String name;

    @ApiModelProperty("年龄")
    @Max(200)
    private Integer age;

    @ApiModelProperty("邮箱")
    @Pattern(regexp = "^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+$")
    private String email;
}

@Data
@Accessors(chain = true)
@ApiModel("用户VO类")
public class PersonVO {

    @ApiModelProperty("用户id")
    private Long id;

    @ApiModelProperty("姓名")
    private String name;

    @ApiModelProperty("年龄")
    private Integer age;

    @ApiModelProperty("邮箱")
    private String email;
}

```

一般来说，生产环境并不需要开启接口文档功能，只需要在生产环境加如下配置即可：

```yaml
springfox:
  documentation:
    enabled: false
```

#### 常用文档注解

**使用文档注解的目的只是为了添加中文说明，仅此而已**！

| 注解| 说明 | 示例 |
|------------|-----------|-----------|
| `@Api` | 标注在`Controller`上，用于给控制器添加中文说明 | @Api(tags = "Person管理") |
| `@ApiOperation` | 标注在方法上，用于给请求添加中文说明 | @ApiOperation("保存用户") |
| `@ApiParam` | 标注在方法参数上，用于给请求参数添加中文说明 | @ApiParam("用户id") |
| `@ApiModel` | 标注在`DTO`实体类上，用于给请求体添加中文说明 | @ApiModel("保存用户DTO类") |
| `@ApiModelProperty` | 标注在`DTO`实体类的字段上，用于给请求体字段添加中文说明 | @ApiModelProperty("姓名") |

#### 示例截图

![swagger-demo](https://chentianming11.github.io/images/spring/swagger-demo.png)

#### 源码

源码地址：[https://github.com/chentianming11/spring-validation](https://github.com/chentianming11/spring-validation)

启动项目，访问[http://localhost:8080/doc.html](http://localhost:8080/doc.html)可查看接口文档。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~
