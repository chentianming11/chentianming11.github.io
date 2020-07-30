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

在《[【spring-boot】项目下最优雅的http客户端工具，用它就够了](https://juejin.im/post/5f14d81cf265da22ee7fbff0)》这篇文章中，我们知道了`retrofit-spring-boot-starter`的使用方式。本篇文章继续继续介绍`retrofit-spring-boot-starter`的实现原理，从零开始介绍**如何在spring-boot项目中基于Retrofit实现自己的轻量级http调用工具**。

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

![retrofit-uml](/images/retrofit/retrofit-uml.png)

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

2. `ClassPathRetrofitClientScanner`

    在`ClassPathRetrofitClientScanner`实现中，**`BeanDefinition`的`beanClass`属性全部设置为了`RetrofitFactoryBean.class`，同时将接口自身的类型传递到了`RetrofitFactoryBean`的`retrofitInterface`属性中**。这说明，最终创建Bean实例是通过`RetrofitFactoryBean`来完成的。

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

这样，我们就完成了创建`HttpService`Bean实例的功能了。在使用的时候直接注入`HttpService`，然后调用其方法就能发送对应的`http`请求。

## 结语

总的来说，在spring-boot项目中基于Retrofit实现自己的轻量级http调用工具的核心只有两点：第一是注册`HttpService`接口的`BeanDefinition`，第二就是构建`Retrofit`来创建`HttpService`的代理对象。如需了解更多细节，建议直接查看[retrofit-spring-boot-starter源码](https://github.com/LianjiaTech/retrofit-spring-boot-starter)。
