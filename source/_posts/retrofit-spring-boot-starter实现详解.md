---
title: 通过Java接口发送http请求【retrofit-spring-boot-starter】实现详解
abbrlink: 1979374763
date: 2020-07-16 17:00:05
tags:
  - http
  - spring
categories:
  - 开源项目
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

## `retrofit`与`spring-boot`框架整合概述

`retrofit`提供了很强大的**通过接口发送`http`请求的功能**，但是美中不足的一点是，`Retrofit`官方没有支持与`spring-boot`框架快速整合。因此，我开发了[retrofit-spring-boot-starter](https://github.com/LianjiaTech/retrofit-spring-boot-starter)，实现了`Retrofit`与`spring-boot`框架快速整合，并且支持了部分功能增强。接下来我会详细介绍`retrofit-spring-boot-starter`的实现细节，以及针对一些常见业务场景的解决方案。

### spring容器初始化

为了能够顺利展开下面的内容，我们先的简单了解一下`spring`容器初始化。简单来讲，`spring`容器初始化主要包含以下2个步骤：

1. **注册Bean定义**：就是扫描并解析**配置文件**或者**某些注解**Bean属性(包括`beanName`、`beanClassName`、`scope`、`isSingleton`等等)得到`BeanDefinition`对象，然后将其注册到`BeanDefinitionRegistry`中。
2. **创建Bean实例**：就是根据`BeanDefinition`信息，创建Bean实例，然后将实例对象保存到`spring`容器中，创建的方式包括反射创建、工厂方法创建和工厂Bean(`FactoryBean`)创建等等。

当然，实际的`spring`容器初始化比这复杂的多，考虑到这块不是本文的重点，暂时这么理解就行。

> 后续也会写spring源码解读的专栏，敬请期待~

## `retrofit-spring-boot-starter`实现概述

相信大家在`spring-boot`项目中都用过`mybatis`，可以认为`mybatis`实现了**通过接口发送sql语句的功能**，并且**接口实例**完全由`spring`容器管理。与之类似，**我们要实现的核心功能就将自定义的`http`接口实例完全交给`spring`容器管理**。为了实现这一目标，我们最重要的是要实现以下2个功能。

1. 注册接口Bean定义
2. 创建接口Bean实例

### 注册接口Bean定义

注册接口Bean定义主要分以下3步完成：

1. **定义`@RetrofitClient`注解**：用来标记用户定义的`http`接口。

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    @Documented
    public @interface RetrofitClient {

        // 还有很多其他属性，这里暂不展开
    }
    ```

2. **定义`@RetrofitScan`注解**：用来指定要扫描的范围。

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

3. **实现`ImportBeanDefinitionRegistrar`接口**：将用指定路径下带有`@RetrofitClient`注解的接口,扫描并注册到`BeanDefinitionRegistry`中。

> `ImportBeanDefinitionRegistrar`实现类可以用来扩展实现`BeanDefinition`注册。

```java
public class RetrofitClientRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, BeanClassLoaderAware {

    // 省略了其它代码

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 扫描指定路径下@RetrofitClient注解的接口，并注册到BeanDefinitionRegistry
    }
}
```

### 创建接口Bean实例

由于我们只定义了`http`接口类，而具体的实现肯定是交由`retrofit`去创建的。因此，我们采用`FactoryBean`的方式来创建`bean`实例(实际上`mybatis`与`spring`整合也是这么实现的)。具体就是实现`FactoryBean<T>`接口，在`getObject()`方法中实现**创建接口Bean实例**的逻辑。

```java
public class RetrofitFactoryBean<T> implements FactoryBean<T>, EnvironmentAware, InitializingBean, ApplicationContextAware {

    // 省略了其它代码

    @Override
    @SuppressWarnings("unchecked")
    public T getObject() throws Exception {
       // 创建接口Bean实例逻辑
    }
}
```

## `retrofit-spring-boot-starter`实现详解

接下来详细介绍`retrofit-spring-boot-starter`实现细节。

### 注册接口Bean定义详解

主要就是将指定路径下带有`@RetrofitClient`注解的接口注册到`BeanDefinitionRegistry`中。需要注意的一点是，我们为了使用`FactoryBean<T>`的方式来创建bean实例，**将`BeanDefinition`的`beanClass`属性全部设置为了`RetrofitFactoryBean.class`，同时将接口自身的类型传递到了`RetrofitFactoryBean`的`retrofitInterface`属性中**。

1. `RetrofitClientRegistrar`实现：

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

2. `ClassPathRetrofitClientScanner`实现：

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

### 创建接口Bean实例详解

我们已经知道使用`Retrofit`对象可以创建接口代理对象，因此**创建接口Bean实例**的关键点就在于如何构建`Retrofit`对象。我们首先看一下`Retrofit`的UML类图(只列出了我们关注的依赖)。

![retrofit-uml](/images/retrofit/retrofit-uml.png)

通过分析UML类图，我们可以发现，构建`Retrofit`对象的时候，可以注入以下4个属性：

1. `HttpUrl`
2. `CallAdapter`
3. `Converter`
4. `OkHttpClient`

而构建`OkHttpClient`对象的时候，可以注入`Interceptor`和`ConnectionPool`属性。

因此**为了构建`Retrofit`对象，我们要先创建`HttpUrl`、`CallAdapter`、`Converter`和`OkHttpClient`；而要构建`OkHttpClient`对象就得先创建`Interceptor`和`ConnectionPool`**。接下来，我们重点了解一下`CallAdapter`和`Converter`，其它的比较简单就不展开了！

#### 调用适配器`CallAdapter`

`Retrofit`本质上是将`http`请求封装成了`Call<T>`对象，后续真正的发送请求和获取响应都是通过这个对象完成的。但是如果每次都要用户自己拿`Call<T>`对象执行操作，灵活性就太差了。因此`Retrofit`提供了调用适配器`CallAdapter`，支持**将`Call<T>`对象适配成其它类型的对象**并返回。**同一个`Retrofit`可以配置多个`CallAdapter`，执行的时候会自动根据方法返回值类型匹配对应的`CallAdapter`**，如果没找到就会报错！

`CallAdapter`实际上是通过`CallAdapter.Factory`实现的，如果用户需要自定义扩展，继承`CallAdapter.Factory`即可。为了方便用户使用，`retrofit-spring-boot-starter`扩展了2种`CallAdapterFactory`实现：

1. `BodyCallAdapterFactory`
    - 默认启用，可通过配置`retrofit.enable-body-call-adapter=false`关闭
    - 同步执行http请求，将响应体内容适配成接口方法的返回值类型实例。
    - 除了`Retrofit.Call<T>`、`Retrofit.Response<T>`、`java.util.concurrent.CompletableFuture<T>`之后，其它返回类型都可以使用该适配器。
2. `ResponseCallAdapterFactory`
    - 默认启用，可通过配置`retrofit.enable-response-call-adapter=false`关闭
    - 同步执行http请求，将响应体内容适配成`Retrofit.Response<T>`返回。
    - 如果方法的返回值类型为`Retrofit.Response<T>`，则可以使用该适配器。

具体代码就不展开了，大家都看到源码里面查看。

#### 数据转码器`Converter`

`Retrofit`使用`Converter` 将`@Body`注解标注的对象转换成请求体，将响应体数据转换成一个`Java`对象。你可以选用以下几种Converter：

- Gson: com.squareup.Retrofit:converter-gson
- Jackson: com.squareup.Retrofit:converter-jackson
- Moshi: com.squareup.Retrofit:converter-moshi
- Protobuf: com.squareup.Retrofit:converter-protobuf
- Wire: com.squareup.Retrofit:converter-wire
- Simple XML: com.squareup.Retrofit:converter-simplexml

**同一个`Retrofit`可以配置多个`Converter`，以适应不同转换场景**。

#### 配置项和`@RetrofitClient`

前面说过，为了构建`Retrofit`对象，我们需要先创建一些依赖对象。为了能够动态化地创建这些对象，我们可以通过一些配置项和`@RetrofitClient`注解(支持不同接口设置不同属性值)实现。

##### `@RetrofitClient`

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

    /**
     * 使用的连接池名称<br>
     * default连接池自动加载，也可以手动配置覆盖默认default连接池属性
     *
     * @return 使用的连接池名称
     */
    String poolName() default "default";

    /**
     * 连接超时，单位为毫秒
     *
     * @return 连接超时时间
     */
    int connectTimeoutMs() default 10_000;

    /**
     * 读取超时，单位为毫秒
     *
     * @return 读取超时时间
     */
    int readTimeoutMs() default 10_000;

    /**
     * 写入超时，单位为毫秒
     *
     * @return 写入超时时间
     */
    int writeTimeoutMs() default 10_000;

    /**
     * 日志打印级别，支持的日志级别参见{@link Level}
     *
     * @return 日志打印级别
     */
    Level logLevel() default Level.INFO;

    /**
     * 日志打印策略，支持的日志打印策略参见{@link BaseLoggingInterceptor.LogStrategy}
     *
     * @return 日志打印策略
     */
    BaseLoggingInterceptor.LogStrategy logStrategy() default BaseLoggingInterceptor.LogStrategy.BASIC;
}
```

各属性用处如下：

1. `baseUrl`：用来创建`Retrofit`的`HttpUrl`，表示该接口下所有请求都适用的**基础url**。
2. `poolName`：该接口下请求使用的连接池的名称，决定了`ConnectionPool`对象的取值。
3. `connectTimeoutMs`/`readTimeoutMs`/`writeTimeoutMs`：用于构建`OkHttpClient`对象。
4. `logLevel`/`logStrategy`：配置该接口下请求的**日志打印级别**和**日志打印策略**，可用来创建日志打印拦截器`Interceptor`。

##### 配置项

在`spring-boot`项目中，配置项属性使用`RetrofitProperties`来接收。主要就是包含各种动态属性的设置，具体含义可以查看[retrofit-spring-boot-starter文档](https://github.com/LianjiaTech/retrofit-spring-boot-starter)。

```java
@ConfigurationProperties(prefix = "retrofit")
public class RetrofitProperties {

    private static final String DEFAULT_POOL = "default";

    /**
     * 连接池配置
     */
    private Map<String, PoolConfig> pool = new LinkedHashMap<>();

    /**
     * 启用 #{@link BodyCallAdapterFactory} 调用适配器
     */
    private boolean enableBodyCallAdapter = true;

    /**
     * 启用 #{@link ResponseCallAdapterFactory} 调用适配器
     */
    private boolean enableResponseCallAdapter = true;

    /**
     * 启用日志打印
     */
    private boolean enableLog = true;

    /**
     * 日志打印拦截器
     */
    private Class<? extends BaseLoggingInterceptor> loggingInterceptor = DefaultLoggingInterceptor.class;

    /**
     * Http异常信息格式化器，用于将request和response格式化为可阅读的String数据，并发到Exception的信息中。
     */
    private Class<? extends BaseHttpExceptionMessageFormatter> httpExceptionMessageFormatter = DefaultHttpExceptionMessageFormatter.class;

    /**
     * 禁用Void返回类型
     */
    private boolean disableVoidReturnType = false;

    public Map<String, PoolConfig> getPool() {
        if (!pool.isEmpty()) {
            return pool;
        }
        pool.put(DEFAULT_POOL, new PoolConfig(5, 300));
        return pool;
    }
    // 省略setter和getter方法
}
```

##### 配置项自动装配

有了配置项，还需要加工才能直接用于构建`Retrofit`对象，因此我们可以使用`RetrofitConfigBean`进行包装，并且自动装配它。

**`RetrofitConfigBean`**：

```java
public class RetrofitConfigBean {

    private final RetrofitProperties retrofitProperties;

    private HttpExceptionMessageFormatterInterceptor httpExceptionMessageFormatterInterceptor;

    private List<CallAdapter.Factory> callAdapterFactories;

    private List<Converter.Factory> converterFactories;

    private Map<String, ConnectionPool> poolRegistry;

    private Collection<BaseGlobalInterceptor> globalInterceptors;

    public RetrofitConfigBean(RetrofitProperties retrofitProperties) {
        this.retrofitProperties = retrofitProperties;
    }
    // 省略getter和setter方法
```

**`RetrofitAutoConfiguration`**：

```java
@Configuration
@EnableConfigurationProperties(RetrofitProperties.class)
public class RetrofitAutoConfiguration implements ApplicationContextAware {

    @Autowired
    private RetrofitProperties retrofitProperties;

    private ApplicationContext applicationContext;

    @Bean
    @ConditionalOnMissingBean
    public RetrofitConfigBean retrofitConfigBean() throws IllegalAccessException, InstantiationException {
        RetrofitConfigBean retrofitConfigBean = new RetrofitConfigBean(retrofitProperties);
        // 初始化连接池
        Map<String, ConnectionPool> poolRegistry = new ConcurrentHashMap<>(4);
        Map<String, PoolConfig> pool = retrofitProperties.getPool();
        if (pool != null) {
            pool.forEach((poolName, poolConfig) -> {
                long keepAliveSecond = poolConfig.getKeepAliveSecond();
                int maxIdleConnections = poolConfig.getMaxIdleConnections();
                ConnectionPool connectionPool = new ConnectionPool(maxIdleConnections, keepAliveSecond, TimeUnit.SECONDS);
                poolRegistry.put(poolName, connectionPool);
            });
        }
        retrofitConfigBean.setPoolRegistry(poolRegistry);

        // 设置Http异常信息格式化器
        Class<? extends BaseHttpExceptionMessageFormatter> httpExceptionMessageFormatterClass = retrofitProperties.getHttpExceptionMessageFormatter();
        BaseHttpExceptionMessageFormatter alarmFormatter = httpExceptionMessageFormatterClass.newInstance();
        HttpExceptionMessageFormatterInterceptor httpExceptionMessageFormatterInterceptor = new HttpExceptionMessageFormatterInterceptor(alarmFormatter);
        retrofitConfigBean.setHttpExceptionMessageFormatterInterceptor(httpExceptionMessageFormatterInterceptor);

        // callAdapterFactory
        List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
        Collection<CallAdapter.Factory> callAdapterFactoryBeans = getBeans(CallAdapter.Factory.class);
        if (!CollectionUtils.isEmpty(callAdapterFactoryBeans)) {
            callAdapterFactories.addAll(callAdapterFactoryBeans);
        }
        if (retrofitProperties.isEnableBodyCallAdapter()) {
            callAdapterFactories.add(new BodyCallAdapterFactory());
        }
        if (retrofitProperties.isEnableResponseCallAdapter()) {
            callAdapterFactories.add(new ResponseCallAdapterFactory());
        }
        retrofitConfigBean.setCallAdapterFactories(callAdapterFactories);

        // converterFactory
        List<Converter.Factory> converterFactories = new ArrayList<>();
        Collection<Converter.Factory> converterFactoryBeans = getBeans(Converter.Factory.class);
        if (!CollectionUtils.isEmpty(converterFactoryBeans)) {
            converterFactories.addAll(converterFactoryBeans);
        }
        JacksonConverterFactory defaultJacksonConverterFactory = JacksonConverterFactory.create(new ObjectMapper()
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                .setSerializationInclusion(JsonInclude.Include.NON_NULL));
        converterFactories.add(defaultJacksonConverterFactory);
        retrofitConfigBean.setConverterFactories(converterFactories);

        // globalInterceptors
        Collection<BaseGlobalInterceptor> globalInterceptors = getBeans(BaseGlobalInterceptor.class);
        retrofitConfigBean.setGlobalInterceptors(globalInterceptors);

        return retrofitConfigBean;
    }

    private <U> Collection<U> getBeans(Class<U> clz) {
        try {
            Map<String, U> beanMap = applicationContext.getBeansOfType(clz);
            return beanMap.values();
        } catch (BeansException e) {
            // do nothing
        }
        return null;
    }


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}

```

#### 使用`RetrofitFactoryBean<T>`来创建接口实例

有了前面各种形式配置数据，通过`RetrofitFactoryBean<T>`就能创建接口实例了。简单来说，创建实例对象主要包含以下步骤：

1. 基于**连接池配置**创建或选取连接池`ConnectionPool`对象；
2. 创建或选取各种各样的拦截器`Interceptor`对象；
3. 基于`ConnectionPool`和`Interceptor`以及一些超时设置创建`OkHttpClient`对象；
4. 基于`OkHttpClient`、`CallAdapter`、`Converter`和`baseUrl`构建`Retrofit`；
5. 基于`Retrofit`构建接口代理对象！

```java
public class RetrofitFactoryBean<T> implements FactoryBean<T>, EnvironmentAware, ApplicationContextAware {

    private Class<T> retrofitInterface;

    private Environment environment;

    private RetrofitProperties retrofitProperties;

    private RetrofitConfigBean retrofitConfigBean;

    private ApplicationContext applicationContext;

    public RetrofitFactoryBean(Class<T> retrofitInterface) {
        this.retrofitInterface = retrofitInterface;
    }

    @Override
    @SuppressWarnings("unchecked")
    public T getObject() throws Exception {
        checkRetrofitInterface(retrofitInterface);
        Retrofit retrofit = getRetrofit(retrofitInterface);
        return retrofit.create(retrofitInterface);
    }

    /**
     * RetrofitInterface检查
     *
     * @param retrofitInterface .
     */
    private void checkRetrofitInterface(Class<T> retrofitInterface) {
        // check class type
        Assert.isTrue(retrofitInterface.isInterface(), "@RetrofitClient只能作用在接口类型上！");
        Method[] methods = retrofitInterface.getMethods();
        for (Method method : methods) {
            Class<?> returnType = method.getReturnType();
            Assert.isTrue(!void.class.isAssignableFrom(returnType),
                    "不支持使用void关键字做返回类型，请使用java.lang.Void! method=" + method);
            if (retrofitProperties.isDisableVoidReturnType()) {
                Assert.isTrue(!Void.class.isAssignableFrom(returnType),
                        "已配置禁用Void作为返回值，请指定其他返回类型！method=" + method);
            }
        }
    }


    @Override
    public Class<T> getObjectType() {
        return this.retrofitInterface;
    }


    @Override
    public boolean isSingleton() {
        return true;
    }


    /**
     * 获取okhttp3连接池
     *
     * @param retrofitClientInterfaceClass retrofitClient接口类
     * @return okhttp3连接池
     */
    private synchronized okhttp3.ConnectionPool getConnectionPool(Class<?> retrofitClientInterfaceClass) {
        RetrofitClient retrofitClient = retrofitClientInterfaceClass.getAnnotation(RetrofitClient.class);
        String poolName = retrofitClient.poolName();
        Map<String, ConnectionPool> poolRegistry = retrofitConfigBean.getPoolRegistry();
        Assert.notNull(poolRegistry, "poolRegistry不存在！请设置retrofitConfigBean.poolRegistry！");
        ConnectionPool connectionPool = poolRegistry.get(poolName);
        Assert.notNull(connectionPool, "当前poolName对应的连接池不存在！poolName = " + poolName);
        return connectionPool;
    }


    /**
     * 获取OkHttpClient实例，一个接口接口对应一个OkHttpClient
     *
     * @param retrofitClientInterfaceClass retrofitClient接口类
     * @return OkHttpClient实例
     */
    private synchronized OkHttpClient getOkHttpClient(Class<?> retrofitClientInterfaceClass) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        okhttp3.ConnectionPool connectionPool = getConnectionPool(retrofitClientInterfaceClass);
        RetrofitClient retrofitClient = retrofitClientInterfaceClass.getAnnotation(RetrofitClient.class);
        // 构建一个OkHttpClient对象
        OkHttpClient.Builder okHttpClientBuilder = new OkHttpClient.Builder()
                .connectTimeout(retrofitClient.connectTimeoutMs(), TimeUnit.MILLISECONDS)
                .readTimeout(retrofitClient.readTimeoutMs(), TimeUnit.MILLISECONDS)
                .writeTimeout(retrofitClient.writeTimeoutMs(), TimeUnit.MILLISECONDS)
                .connectionPool(connectionPool);
        // 添加接口上注解定义的拦截器
        List<Interceptor> interceptors = new ArrayList<>(findInterceptorByAnnotation(retrofitClientInterfaceClass));
        // 添加全局拦截器
        Collection<BaseGlobalInterceptor> globalInterceptors = retrofitConfigBean.getGlobalInterceptors();
        if (!CollectionUtils.isEmpty(globalInterceptors)) {
            interceptors.addAll(globalInterceptors);
        }
        interceptors.forEach(okHttpClientBuilder::addInterceptor);

        // 日志打印拦截器
        if (retrofitProperties.isEnableLog()) {
            Class<? extends BaseLoggingInterceptor> loggingInterceptorClass = retrofitProperties.getLoggingInterceptor();
            Constructor<? extends BaseLoggingInterceptor> constructor = loggingInterceptorClass.getConstructor(Level.class, BaseLoggingInterceptor.LogStrategy.class);
            BaseLoggingInterceptor loggingInterceptor = constructor.newInstance(retrofitClient.logLevel(), retrofitClient.logStrategy());
            okHttpClientBuilder.addInterceptor(loggingInterceptor);
        }
        //  http异常信息格式化
        HttpExceptionMessageFormatterInterceptor httpExceptionMessageFormatterInterceptor = retrofitConfigBean.getHttpExceptionMessageFormatterInterceptor();
        if (httpExceptionMessageFormatterInterceptor != null) {
            okHttpClientBuilder.addInterceptor(httpExceptionMessageFormatterInterceptor);
        }
        return okHttpClientBuilder.build();
    }


    /**
     * 获取retrofitClient接口类上定义的拦截器集合
     *
     * @param retrofitClientInterfaceClass retrofitClient接口类
     * @return 拦截器实例集合
     */
    @SuppressWarnings("unchecked")
    private List<Interceptor> findInterceptorByAnnotation(Class<?> retrofitClientInterfaceClass) throws InstantiationException, IllegalAccessException {
        Annotation[] classAnnotations = retrofitClientInterfaceClass.getAnnotations();
        List<Interceptor> interceptors = new ArrayList<>();
        // 找出被@InterceptMark标记的注解
        List<Annotation> interceptAnnotations = new ArrayList<>();
        for (Annotation classAnnotation : classAnnotations) {
            Class<? extends Annotation> annotationType = classAnnotation.annotationType();
            if (annotationType.isAnnotationPresent(InterceptMark.class)) {
                interceptAnnotations.add(classAnnotation);
            }
        }
        for (Annotation interceptAnnotation : interceptAnnotations) {
            // 获取注解属性数据
            Map<String, Object> annotationAttributes = AnnotationUtils.getAnnotationAttributes(interceptAnnotation);
            Object handler = annotationAttributes.get("handler");
            Assert.notNull(handler, "@InterceptMark标记的注解必须配置: Class<? extends BasePathMatchInterceptor> handler()");
            Assert.notNull(annotationAttributes.get("include"), "@InterceptMark标记的注解必须配置: String[] include()");
            Assert.notNull(annotationAttributes.get("exclude"), "@InterceptMark标记的注解必须配置: String[] exclude()");
            Class<? extends BasePathMatchInterceptor> interceptorClass = (Class<? extends BasePathMatchInterceptor>) handler;
            BasePathMatchInterceptor interceptor = getInterceptorInstance(interceptorClass);
            Map<String, Object> annotationResolveAttributes = new HashMap<>(8);
            // 占位符属性替换
            annotationAttributes.forEach((key, value) -> {
                if (value instanceof String) {
                    String newValue = environment.resolvePlaceholders((String) value);
                    annotationResolveAttributes.put(key, newValue);
                } else {
                    annotationResolveAttributes.put(key, value);
                }
            });
            // 动态设置属性值
            BeanExtendUtils.populate(interceptor, annotationResolveAttributes);
            interceptors.add(interceptor);
        }
        return interceptors;
    }

    /**
     * 获取路径拦截器实例，优先从spring容器中取。如果spring容器中不存在，则无参构造器实例化一个
     *
     * @param interceptorClass 路径拦截器类的子类，参见@{@link BasePathMatchInterceptor}
     * @return 路径拦截器实例
     */
    private BasePathMatchInterceptor getInterceptorInstance(Class<? extends BasePathMatchInterceptor> interceptorClass) throws IllegalAccessException, InstantiationException {
        // spring bean
        try {
            return applicationContext.getBean(interceptorClass);
        } catch (BeansException e) {
            // spring容器获取失败，反射创建
            return interceptorClass.newInstance();
        }
    }


    /**
     * 获取Retrofit实例，一个retrofitClient接口对应一个Retrofit实例
     *
     * @param retrofitClientInterfaceClass retrofitClient接口类
     * @return Retrofit实例
     */
    private synchronized Retrofit getRetrofit(Class<?> retrofitClientInterfaceClass) throws InstantiationException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        RetrofitClient retrofitClient = retrofitClientInterfaceClass.getAnnotation(RetrofitClient.class);
        String baseUrl = retrofitClient.baseUrl();
        // 解析baseUrl占位符
        baseUrl = environment.resolveRequiredPlaceholders(baseUrl);
        OkHttpClient client = getOkHttpClient(retrofitClientInterfaceClass);
        Retrofit.Builder retrofitBuilder = new Retrofit.Builder()
                .baseUrl(baseUrl)
                .client(client);
        // 添加CallAdapter.Factory
        List<CallAdapter.Factory> callAdapterFactories = retrofitConfigBean.getCallAdapterFactories();
        if (!CollectionUtils.isEmpty(callAdapterFactories)) {
            callAdapterFactories.forEach(retrofitBuilder::addCallAdapterFactory);
        }
        // 添加Converter.Factory
        List<Converter.Factory> converterFactories = retrofitConfigBean.getConverterFactories();
        if (!CollectionUtils.isEmpty(converterFactories)) {
            converterFactories.forEach(retrofitBuilder::addConverterFactory);
        }
        return retrofitBuilder.build();
    }


    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
        this.retrofitConfigBean = applicationContext.getBean(RetrofitConfigBean.class);
        this.retrofitProperties = retrofitConfigBean.getRetrofitProperties();
    }
}

```

### 注解式拦截器实现详解


