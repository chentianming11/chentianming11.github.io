---
title: 手把手教你基于Retrofit实现自己的轻量级http调用工具
abbrlink: 1979374763
date: 2020-07-16 17:00:05
tags:
  - http
  - spring
categories:
  - 开源项目
---

在《[spring-boot项目整合Retrofit最佳实践，最优雅的HTTP客户端工具！](https://juejin.im/post/5f14d81cf265da22ee7fbff0)》这篇文章中，我们知道了`retrofit-spring-boot-starter`的使用方式。本篇文章继续继续介绍`retrofit-spring-boot-starter`的实现原理，从零开始介绍**如何在spring-boot项目中基于Retrofit实现自己的轻量级http调用工具**。

> 项目源码：[retrofit-spring-boot-starter](https://github.com/LianjiaTech/retrofit-spring-boot-starter)

<!--more-->

## 确定实现思路

我们首先直接看一下使用`retrofit`原始API是如何发起一个http请求的。

1. 定义接口

    ```java
    public interface GitHubService {
    @GET("users/{user}/repos")
    Call<List<Repo>> listRepos(@Path("user") String user);
    }
    ```

2. 创建接口代理对象

    ```java
    Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com/")
        .build();

    // 实际业务场景构建Retrofit比这复杂多了，这里最简单化处理

    GitHubService service = retrofit.create(GitHubService.class);
    ```

3. 发起请求

    ```java
    Call<List<Repo>> repos = service.listRepos("octocat");
    ```

可以看到，`Retrofit`本身已经很好的支持了通过接口发起`htp`请求。但是如果我们项目每一个业务代码都要写上面的样板代码，会非常的繁琐。有没有一种方式**让用户只关注接口定义，其它事情全部交给框架自动处理**？这个时候我们可能会联想到`spring-boot`项目下使用`Mybatis`，用户只需要定义`Mapper`接口和书写`sql`即可，完全不用管与`JDBC`的交互细节。与之类似，**我们最终也要实现让用户只需要定义`HttpService`接口，不用管其他底层实现细节**。

## 相关知识介绍

为了方便后面的介绍，我们先得了解一下几个相关知识点。

### spring容器初始化

我们首先要简单了解一下`spring`容器初始化。简单来讲，`spring`容器初始化主要包含以下2个步骤：

1. **注册Bean定义**：扫描并解析**配置文件**或者**某些注解**得到Bean属性(包括`beanName`、`beanClassName`、`scope`、`isSingleton`等等)，然后基于这个`bean`属性创建`BeanDefinition`对象，最后将其注册到`BeanDefinitionRegistry`中。
2. **创建Bean实例**：根据`BeanDefinitionRegistry`里面的`BeanDefinition`信息，创建Bean实例，并将实例对象保存到`spring`容器中，创建的方式包括反射创建、工厂方法创建和工厂Bean(`FactoryBean`)创建等等。

当然，实际的`spring`容器初始化比这复杂的多，考虑到这块不是本文的重点，暂时这么理解就行。

### `Retrofit`对象简介

我们已经知道使用`Retrofit`对象可以创建接口代理对象，接下来看一下`Retrofit`的UML类图(只列出了我们关注的依赖)：

![retrofit-uml](https://chentianming11.github.io/images/retrofit/retrofit-uml.png)

通过分析UML类图，我们可以发现，构建`Retrofit`对象的时候，可以注入以下4个属性：

1. `HttpUrl`：`http`请求的`baseUrl`。
2. `CallAdapter`：将`Call<T>`适配为接口方法返回值类型。
3. `Converter`：将`@Body`标记的方法参数序列化为请求体数据；将响应体数据反序列化为响应对象。
4. `OkHttpClient`：底层发送`http`请求的客户端对象。

而构建`OkHttpClient`对象的时候，可以注入`Interceptor`(请求拦截器)和`ConnectionPool`(连接池)属性。

因此**为了构建`Retrofit`对象，我们要先创建`HttpUrl`、`CallAdapter`、`Converter`和`OkHttpClient`；而要构建`OkHttpClient`对象就得先创建`Interceptor`和`ConnectionPool`**。

## 实现详解

### 注册Bean定义

为了实现将`HttpService`接口代理对象完全交由`spring`容器管理，首先就得将`HttpService`接口扫描并注册到`BeanDefinitionRegistry`中。`spring`提供了`ImportBeanDefinitionRegistrar`接口，支持了自定义注册`BeanDefinition`的功能。因此我们先定义`RetrofitClientRegistrar`类用来实现上述功能。具体实现如下：

1. `RetrofitClientRegistrar`

    `RetrofitClientRegistrar`从`@RetrofitScan`注解中提取出要扫描的基础包路径之后，将具体的扫描注册逻辑交给了`ClassPathRetrofitClientScanner`处理。

    ```java
    public class RetrofitClientRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, BeanClassLoaderAware {

        // 省略其它代码

        @Override
        public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            AnnotationAttributes attributes = AnnotationAttributes
                    .fromMap(metadata.getAnnotationAttributes(RetrofitScan.class.getName()));
            // 扫描指定路径下@RetrofitClient注解的接口，并注册到BeanDefinitionRegistry
            // 真正的扫描注册逻辑交给了ClassPathRetrofitClientScanner执行
            ClassPathRetrofitClientScanner scanner = new ClassPathRetrofitClientScanner(registry, classLoader);
            if (resourceLoader != null) {
                scanner.setResourceLoader(resourceLoader);
            }
            //指定扫描的基础包
            String[] basePackages = getPackagesToScan(attributes);
            scanner.registerFilters();
            // 扫描并注册到BeanDefinition
            scanner.doScan(basePackages);
        }
    }
    ```

2. `ClassPathRetrofitClientScanner`

    `ClassPathRetrofitClientScanner`继承了`ClassPathBeanDefinitionScanner`，这是Spring提供的类路径下`BeanDefinition`的扫描器。需要注意的一点是：**`BeanDefinition`的`beanClass`属性全部设置为了`RetrofitFactoryBean.class`，同时将接口自身的类型传递到了`RetrofitFactoryBean`的`retrofitInterface`属性中**。这说明，最终创建Bean实例是通过`RetrofitFactoryBean`来完成的。

    ```java
    public class ClassPathRetrofitClientScanner extends ClassPathBeanDefinitionScanner {

        // 省略其它代码

        private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
            GenericBeanDefinition definition;
            for (BeanDefinitionHolder holder : beanDefinitions) {
                definition = (GenericBeanDefinition) holder.getBeanDefinition();
                if (logger.isDebugEnabled()) {
                    logger.debug("Creating RetrofitClientBean with name '" + holder.getBeanName()
                            + "' and '" + definition.getBeanClassName() + "' Interface");
                }
                definition.getConstructorArgumentValues().addGenericArgumentValue(Objects.requireNonNull(definition.getBeanClassName()));
                // beanClass全部设置为RetrofitFactoryBean
                definition.setBeanClass(RetrofitFactoryBean.class);
            }
        }
    }
    ```

这样，我们就完成了**扫描指定路径下带有`@RetrofitClient`注解的接口，并将其注册到`BeanDefinitionRegistry`的功能了**。

> `@RetrofitClient`注解要标识在`HttpService`的接口上！`@RetrofitScan`指定了要扫描的包路径。具体可看考源码。

### 创建Bean实例

上面已经说了创建Bean实例实际上是通过`RetrofitFactoryBean`实现的。具体就是实现`FactoryBean<T>`接口，然后重写`getObject()`方法来完成**创建接口Bean实例**的逻辑。并且，我们也已经知道通过`Retrofit`对象能够生成接口代理对象。因此`getObject()`方法的核心就是构建`Retrofit`对象，并基于此生成`http`接口代理对象。

1. 配置项和`@RetrofitClient`
   为了更加灵活的构建`Retrofit`对象，我们可以通过配置项以及`@RetrofitClient`注解属性传递一些动态参数信息。`@RetrofitClient`包含的属性如下：
    1. `baseUrl`：用来创建`Retrofit`的`HttpUrl`，表示该接口下所有请求都适用的`基础url`。
    2. `poolName`：该接口下请求使用的连接池的名称，决定了`ConnectionPool`对象的取值。
    3. `connectTimeoutMs/readTimeoutMs/writeTimeoutMs`：用于构建`OkHttpClien`t对象的超时时间设置。
    4. `logLevel/logStrategy`：配置该接口下请求的日志打印级别和日志打印策略，可用来创建日志打印拦截器`Interceptor`。

2. `RetrofitFactoryBean`
    `RetrofitFactoryBean`实现逻辑非常复杂，概括起来主要包含以下几点：
   1. 通过配置项数据以及`@RetrofitClient`注解数据完成了`Retrofit`对象的构建。
   2. 每一个`HttpService`接口就会构建一个`Retrofit`对象，每一个`Retrofit`对象就会构建对应的`OkHttpClient`对象。
   3. 可扩展的注解式拦截器是通过`InterceptMark`注解标记实现的，路径拦截匹配是通过`BasePathMatchInterceptor`实现的。

    ```java
    public class RetrofitFactoryBean<T> implements FactoryBean<T>, EnvironmentAware, ApplicationContextAware {

        // 省略其它代码

        public RetrofitFactoryBean(Class<T> retrofitInterface) {
            this.retrofitInterface = retrofitInterface;
        }

        @Override
        @SuppressWarnings("unchecked")
        public T getObject() throws Exception {
            // 接口校验
            checkRetrofitInterface(retrofitInterface);
            // 构建Retrofit对象
            Retrofit retrofit = getRetrofit(retrofitInterface);
            // 基于Retrofit创建接口代理对象
            return retrofit.create(retrofitInterface);
        }

        /**
        * 获取OkHttpClient实例，一个接口接口对应一个OkHttpClient
        *
        * @param retrofitClientInterfaceClass retrofitClient接口类
        * @return OkHttpClient实例
        */
        private synchronized OkHttpClient getOkHttpClient(Class<?> retrofitClientInterfaceClass) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
            // 基于各种条件构建OkHttpClient
        }


        /**
        * 获取Retrofit实例，一个retrofitClient接口对应一个Retrofit实例
        *
        * @param retrofitClientInterfaceClass retrofitClient接口类
        * @return Retrofit实例
        */
        private synchronized Retrofit getRetrofit(Class<?> retrofitClientInterfaceClass) throws InstantiationException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
            // 构建retrofit
        }
    ```

这样，我们就完成了创建`HttpService`Bean实例的功能了。在使用的时候直接注入`HttpService`，然后调用其方法就能发送对应的`http`请求。

## 结语

总的来说，在spring-boot项目中基于Retrofit实现自己的轻量级http调用工具的核心只有两点：第一是注册`HttpService`接口的`BeanDefinition`，第二就是构建`Retrofit`来创建`HttpService`的代理对象。如需了解更多细节，建议直接查看[retrofit-spring-boot-starter源码](https://github.com/LianjiaTech/retrofit-spring-boot-starter)。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~
