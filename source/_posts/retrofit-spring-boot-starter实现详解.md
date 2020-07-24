---
title: 基于Retrofit实现spring-boot项目下轻量级http调用框架
abbrlink: 1979374763
date: 2020-07-16 17:00:05
tags:
  - http
  - spring
categories:
  - 开源项目
---

在[【spring-boot】项目下最优雅的http客户端工具，用它就够了](https://chentianming11.github.io/posts/2257172027/)这篇文章中，我们知道了`retrofit-spring-boot-starter`的使用方式。本篇文章继续继续介绍`retrofit-spring-boot-starter`的实现原理，从零开始介绍**如何基于Retrofit实现spring-boot项目下轻量级http调用框架**。

> 建议先阅读[【spring-boot】项目下最优雅的http客户端工具，用它就够了](https://chentianming11.github.io/posts/2257172027/)，再看本篇文章。

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

可以看到，基于`Retrofit`本身已经很好的支持了通过接口发起`htp`请求。但是如果我们项目每一个业务代码都会写上面的样板代码，会非常的繁琐和麻烦。有没有一种方式让用户只关注接口定义，其它事情全部交给框架自动处理？这个时候我们可能会联想到`spring-boot`项目下使用`Mybatis`，用户只需要定义`Mapper`接口和书写`sql`即可，完全不用管与`JDBC`的交互细节。与之类似，**我们也要实现让用户只需要定义`HttpService`接口，不用管其他底层实现细节**。

## 相关知识介绍

为了方便后面介绍实现过程，我们先得了解一下几个相关知识点。

### spring容器初始化

我们首先要简单了解一下`spring`容器初始化。简单来讲，`spring`容器初始化主要包含以下2个步骤：

1. **注册Bean定义**：扫描并解析**配置文件**或者**某些注解**得到Bean属性(包括`beanName`、`beanClassName`、`scope`、`isSingleton`等等)，然后基于这个`bean`属性创建`BeanDefinition`对象，最后将其注册到`BeanDefinitionRegistry`中。
2. **创建Bean实例**：根据`BeanDefinitionRegistry`里面的`BeanDefinition`信息，创建Bean实例，并将实例对象保存到`spring`容器中，创建的方式包括反射创建、工厂方法创建和工厂Bean(`FactoryBean`)创建等等。

当然，实际的`spring`容器初始化比这复杂的多，考虑到这块不是本文的重点，暂时这么理解就行。

### `Retrofit`对象简介

我们已经知道使用`Retrofit`对象可以创建接口代理对象。我们看一下`Retrofit`的UML类图(只列出了我们关注的依赖)：

![retrofit-uml](/images/retrofit/retrofit-uml.png)

通过分析UML类图，我们可以发现，构建`Retrofit`对象的时候，可以注入以下4个属性：

1. `HttpUrl`：`http`请求的`baseUrl`。
2. `CallAdapter`：将`Call<T>`适配为接口方法返回值类型。
3. `Converter`：将`@Body`标记的方法参数序列化为请求体数据；将响应体数据反序列化为响应对象。
4. `OkHttpClient`：底层发送`http`请求的客户端对象。

而构建`OkHttpClient`对象的时候，可以注入`Interceptor`(请求拦截器)和`ConnectionPool`(连接池)属性。

因此**为了构建`Retrofit`对象，我们要先创建`HttpUrl`、`CallAdapter`、`Converter`和`OkHttpClient`；而要构建`OkHttpClient`对象就得先创建`Interceptor`和`ConnectionPool`**。

**未完待续，后面是文章素材**。

## 实现概述

### 注册接口Bean定义

通过**实现`ImportBeanDefinitionRegistrar`接口**可以实现自定义注册Bean定义的功能。

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

创建接口Bean实例可以采用`FactoryBean`的方式实现(实际上`mybatis`与`spring`整合也是这么实现的)。具体就是实现`FactoryBean<T>`接口，在`getObject()`方法中实现**创建接口Bean实例**的逻辑。

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



## 实现详解

### `@RetrofitScan`

我们知道，在使用`retrofit-spring-boot-starter`的时候，我们在配置类上加了`@RetrofitScan`注解。可以认为，这就是注册接口Bean定义的入口。`@RetrofitScan`主要作用只有以下2点：

1. 指定了要扫描的包路径
2. 具体`BeanDefinition`注册逻辑由`RetrofitClientRegistrar`完成。

具体代码如下：

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

### `RetrofitClientRegistrar`

`RetrofitClientRegistrar`从`@RetrofitScan`注解中提取出要扫描的基础包路径之后，将具体的扫描注册逻辑交给了`ClassPathRetrofitClientScanner`处理，具体代码如下：

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

我们接着看`ClassPathRetrofitClientScanner`的实现。在`ClassPathRetrofitClientScanner`实现中，**`BeanDefinition`的`beanClass`属性全部设置为了`RetrofitFactoryBean.class`，同时将接口自身的类型传递到了`RetrofitFactoryBean`的`retrofitInterface`属性中**。这说明，最终创建Bean实例是通过`RetrofitFactoryBean`来完成的。

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

### `RetrofitFactoryBean`

`RetrofitFactoryBean`实现逻辑非常复杂的，重要信息概括起来包含以下几点：

1. 通过`RetrofitConfigBean`的配置项数据以及`@RetrofitClient`注解数据完成了`Retrofit`对象的构建。
2. 每一个Java接口就会构建一个`Retrofit`对象，每一个`Retrofit`对象就会构建对应的`OkHttpClient`对象。
3. 可扩展的注解式拦截器是通过`InterceptMark`注解标记实现的，路径拦截匹配是`BasePathMatchInterceptor`实现的。

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
```

### `@RetrofitClient`

`@RetrofitClient`用于标记发送`http`请求的`Java`接口，同时为构建`Retrofit`对象提供基础数据。各属性含义如下：

1. `baseUrl`：用来创建`Retrofit`的`HttpUrl`，表示该接口下所有请求都适用的**基础url**。
2. `poolName`：该接口下请求使用的连接池的名称，决定了`ConnectionPool`对象的取值。
3. `connectTimeoutMs`/`readTimeoutMs`/`writeTimeoutMs`：用于构建`OkHttpClient`对象的超时时间设置。
4. `logLevel`/`logStrategy`：配置该接口下请求的**日志打印级别**和**日志打印策略**，可用来创建日志打印拦截器`Interceptor`。

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

### `RetrofitAutoConfiguration`

`RetrofitAutoConfiguration`是`retrofit-spring-boot-starter`的配置类，主要完成了基于配置数据`RetrofitProperties`构建`RetrofitConfigBean`的功能。而`RetrofitConfigBean`就是`RetrofitFactoryBean`中构建`Retrofit`对象的关键。具体实现的功能如下：

1. 初始化连接池数据
2. 设置`Http`异常信息格式化器
3. 设置`callAdapterFactory`
4. 设置`converterFactory`
5. 设置`globalInterceptors`

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

## 结语

总的来说，`retrofit-spring-boot-starter`主要通过`RetrofitClientRegistrar`完成Java接口的Bean定义注册，通过`RetrofitFactoryBean`完成了Bean实例的创建，从而实现了`retrofit`与`spring-boot`框架的深度整合。如需了解更多细节，建议直接查看[retrofit-spring-boot-starter源码](https://github.com/LianjiaTech/retrofit-spring-boot-starter)。
