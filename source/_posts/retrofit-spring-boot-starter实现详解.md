---
title: 还在使用HttpUtil？快来试试接口化发送http请求！
abbrlink: 1979374763
date: 2020-07-13 17:00:05
tags:
categories:
---

> 在以前的项目中，充斥着各种使用*`okhttp`封装的`HttpUtil`*来发送`http`请求的代码，**这种方式不仅不够灵活，而且也不利于集中管理项目的`http`请求**。

大家都知道`okhttp`是一款由[square](https://square.github.io/)公司开源的`java`版本`http`客户端工具。实际上，[square](https://square.github.io/)公司还开源了基于`okhttp`进一步封装的[retrofit](http://square.github.io/retrofit/)工具，用来支持通过`接口`的方式发起`http`请求。本文首先会介绍`retrofit`的基本使用，然后重点介绍`retrofit`与`spring-boot`框架整合。

<!--more-->

项目源码地址：https://github.com/LianjiaTech/retrofit-spring-boot-starter
**上述项目已在生产环境使用，比较稳定，可以直接引入使用**。

## `retrofit`基本使用

首先要在项目中引入`retrofit`依赖：

```xml
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>retrofit</artifactId>
    <version>2.9.0</version>
</dependency>
```

实际上，`retrofit`基本使用非常简单，主要分为以下3步：

1. 定义接口
2. 构建`retrofit`对象和创建接口代理对象
3. 通过接口发送请求

### 定义接口

首先要定义一个发起`http`请求的接口，**接口下面的一个方法就代表了一个`http`请求**。方法上可以使用不同的`注解`来标识请求的属性和数据，比如请求的方式、参数等等信息。比如：

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

### 构建`retrofit`对象和创建接口代理对象

在定义好接口之后，接下来就是构建`retrofit`对象，并且基于`retrofit`对象创建接口代理对象。

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

### 通过接口发送请求

有了接口代理对象之后，直接调用接口方法就是发送请求了。

```java
Call<List<Repo>> repos = service.listRepos("octocat");
```

`retrofit`基本使用就先介绍到这了，更多的功能特性会在**`retrofit`与`spring-boot`框架整合**部分详细讲解。

## `retrofit`与`spring-boot`框架整合

`retrofit`提供了很强大的**通过接口化发送`http`请求的功能**，但是美中不足的一点是，`Retrofit2`官方没有支持与`spring-boot`框架快速整合。因此，我开发了[retrofit-spring-boot-starter](https://github.com/LianjiaTech/retrofit-spring-boot-starter)，实现了`Retrofit`与`spring-boot`框架快速整合，并且支持了部分功能增强。接下来我会详细介绍`retrofit-spring-boot-starter`的实现细节，以及针对一些常见业务场景的解决方案。

### spring容器初始化

为了能够顺利展开下面的内容，我们先的简单了解一下`spring`容器初始化。简单来讲，`spring`容器初始化主要包含以下2个步骤：

1. **注册Bean定义**：就是扫描并解析**配置文件**或者**某些注解**Bean属性(包括`beanName`、`beanClassName`、`scope`、`isSingleton`等等)得到`BeanDefinition`对象，然后将其注册到`BeanDefinitionRegistry`中。
2. **创建Bean实例**：就是根据`BeanDefinition`信息，创建Bean实例，然后将实例对象保存到`spring`容器中，创建的方式包括反射创建、工厂方法创建和工厂Bean(`FactoryBean`)创建等等。

当然，实际的`spring`容器初始化比这复杂的多，考虑到这块不是本文的重点，暂时这么理解就行。

> 后续也会写spring源码解读的专栏，敬请期待~

### `retrofit-spring-boot-starter`骨架实现

相信大家在`spring-boot`项目中都用过`mybatis`，可以认为`mybatis`实现了**通过接口发送sql语句的功能**，并且**接口实例**完全由`spring`容器管理。与之类似，**我们要实现的核心功能就将自定义的`http`接口实例完全交给`spring`容器管理**。为了实现这一目标，我们最重要的是要实现以下2个功能。

1. 注册接口Bean定义
2. 创建接口Bean实例

#### 注册接口Bean定义

为了实现`spring`将用户定义的`http`接口扫描并注册到`BeanDefinitionRegistry`，我们首先定义了`@RetrofitClient`注解去标记要注册的接口，然后定义了`@RetrofitScan`去指定要扫描的范围，最后通过`RetrofitClientRegistrar`完成了扫描和注册。实现如下：

**@RetrofitClient**：用来标记用户定义的`http`接口。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface RetrofitClient {

    /**
     * 基础url, 支持占位符形式配置。<br>
     * 例如：http://${baseUrl.test}
     *
     * @return RetrofitClient的baseUrl
     */
    String baseUrl();

    // 还有很多其他属性，这里暂不展开
}
```

**RetrofitScan**：用来指定要扫描的范围。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(RetrofitClientRegistrar.class)
public @interface RetrofitScan {

    /**
     * <p>扫描包路径</p>
     * 与basePackages含义相同
     *
     * @return 扫描包路径
     */
    String[] value() default {};


    /**
     * <p>扫描包路径</p>
     * 与value含义相同
     *
     * @return 扫描包路径
     */
    String[] basePackages() default {};

    /**
     * 扫描的classes
     *
     * @return 扫描的classes
     */
    Class<?>[] basePackageClasses() default {};
}
```

**RetrofitClientRegistrar**：扫描并注册到`BeanDefinitionRegistry`。

```java
public class RetrofitClientRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, BeanClassLoaderAware {

    private ResourceLoader resourceLoader;

    private ClassLoader classLoader;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        AnnotationAttributes attributes = AnnotationAttributes
                .fromMap(metadata.getAnnotationAttributes(RetrofitScan.class.getName()));
        // 扫描指定路径下@RetrofitClient注解的接口，并注册到BeanDefinitionRegistry
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

    /**
     * 获取扫描的基础包路径
     *
     * @return 基础包路径
     */
    private String[] getPackagesToScan(AnnotationAttributes attributes) {
        String[] value = attributes.getStringArray("value");
        String[] basePackages = attributes.getStringArray("basePackages");
        Class<?>[] basePackageClasses = attributes.getClassArray("basePackageClasses");
        if (!ObjectUtils.isEmpty(value)) {
            Assert.state(ObjectUtils.isEmpty(basePackages),
                    "@RetrofitScan basePackages and value attributes are mutually exclusive");
        }
        Set<String> packagesToScan = new LinkedHashSet<>();
        packagesToScan.addAll(Arrays.asList(value));
        packagesToScan.addAll(Arrays.asList(basePackages));
        for (Class<?> basePackageClass : basePackageClasses) {
            packagesToScan.add(ClassUtils.getPackageName(basePackageClass));
        }
        return packagesToScan.toArray(new String[packagesToScan.size()]);
    }


    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }


    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }
}
```

**ClassPathRetrofitClientScanner**：具体的扫描器实现。

```java
public class ClassPathRetrofitClientScanner extends ClassPathBeanDefinitionScanner {

    private final ClassLoader classLoader;

    private final static Logger logger = LoggerFactory.getLogger(ClassPathRetrofitClientScanner.class);

    public ClassPathRetrofitClientScanner(BeanDefinitionRegistry registry, ClassLoader classLoader) {
        super(registry, false);
        this.classLoader = classLoader;
    }

    public void registerFilters() {
        AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(RetrofitClient.class);
        this.addIncludeFilter(annotationTypeFilter);
    }


    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
        if (beanDefinitions.isEmpty()) {
            logger.warn("No RetrofitClient was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
        } else {
            processBeanDefinitions(beanDefinitions);
        }
        return beanDefinitions;
    }

    @Override
    protected boolean isCandidateComponent(
            AnnotatedBeanDefinition beanDefinition) {
        if (beanDefinition.getMetadata().isInterface()) {
            try {
                Class<?> target = ClassUtils.forName(
                        beanDefinition.getMetadata().getClassName(),
                        classLoader);
                return !target.isAnnotation();
            } catch (Exception ex) {
                logger.error("load class exception:", ex);
            }
        }
        return false;
    }


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

至此，我们就完成了接口Bean定义的注册。

#### 创建接口Bean实例


