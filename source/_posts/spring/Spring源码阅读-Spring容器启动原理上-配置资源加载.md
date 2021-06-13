---
title: 【Spring源码阅读】Spring容器启动原理(上)-配置资源加载和注册
tags: spring
categories: spring
abbrlink: 468538983
date: 2021-06-13 10:24:45
---

Spring是日常工作中最常用的框架，而`IoC`是其最核心的思想之一。**IoC(Inversion of Control)的意思是控制反转，就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现**。整个容器的启动过程主要包含两大部分，第一部分是配置资源加载和注册，第二部分是`Bean`实例的创建和依赖注入(DI)。本文会根据源码详细介绍配置资源加载和注册的实现原理。

<!--more-->

## Spring核心容器类

为了方便后续介绍，我们先得了解两个Spring核心容器类，分别是`BeanFactory`、`BeanDefinition`。

### BeanFactory

**`BeanFactory`意思是管理`Bean`的工厂，它定义了`getBean()`、`containsBean()`等管理Bean的通用方法，我们熟悉的`ApplicationContext`
接口就继承了`BeanFactory`接口。 可以简单认为，`BeanFactory`实例就是Spring底层容器，它是容器中所有`Bean`的管理者**。

```java
public interface BeanFactory {

	//对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，
	//如果需要得到工厂本身，需要转义
	String FACTORY_BEAN_PREFIX = "&";

	
	//根据bean的名字，获取在IOC容器中得到bean实例
	Object getBean(String name) throws BeansException;

	//根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。
	<T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;


	//提供对bean的检索，看看是否在IOC容器有这个名字的bean
	boolean containsBean(String name);

	// 是否是单例bean
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	// 是否是原型Bean
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
	
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	
	boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
	
	//得到bean实例的Class类型
	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
	
	//得到bean的别名，如果根据别名检索，那么其原名也会被检索出来
	String[] getAliases(String name);

}
```

### BeanDefinition

**`BeanDefinition`意思是`Bean`定义，它描述了`Bean`的各种基础属性信息，比如`beanClassName`、`singleton`、`lazyInit`、`scope`等等。可以认为，`BeanDefinition`是完整创建`Bean`实例的基础，而本文真正介绍的就是`BeanDefinition`的加载和注册原理**。

![BeanDefinition-method](https://chentianming11.github.io/images/spring/ioc/BeanDefinition-method.png)

## 基于Xml的配置资源加载和注册

配置资源加载和注册具体来说就是`BeanDefinition`的定位、加载和注册三部分。在Spring中，容器最后是以`ApplicationContext`实例存在的。根据具体资源的不同，分为各类`ApplicationContext`，比如`ClasspathXmlApplicationContext`，`AnnotationConfigApplicationContext`等等。类图如下：
![ApplicationContext](https://chentianming11.github.io/images/spring/ioc/ApplicationContext.png)

`ApplicationContext`允许上下文嵌套，对于一个`Bean`的查找，优先从当前上下文查找，如果没找到，则从父上下文查找，逐级向上。

### 寻找入口

对于xml配置的Spring应用，在`main()`方法中实例化`ClasspathXmlApplicationContext`即可创建一个IoC容器。我们可以从这个构造方法开始，探究一下IoC容器的初始化过程。

```java
ApplicationContext app = new ClassPathXmlApplicationContext("application.xml");
```

实际调用的构造方法如下：
![ClassPathXmlApplicationContext-constructor](https://chentianming11.github.io/images/spring/ioc/ClassPathXmlApplicationContext-constructor.png)

### 设置资源解析器和配置路径

在创建`ClassPathXmlApplicationContext`容器时，构造方法做以下两项重要工作:

1. 调用父类容器的构造方法`super(parent)`方法为容器设置好`Bean`资源加载器。
2. 调用父类`AbstractRefreshableConfigApplicationContext`的`setConfigLocations(configLocations)`方法设置`Bean`配置路径。

通过追踪`ClassPathXmlApplicationContext`的继承体系，发现其父类的父类`AbstractApplicationContext`中初始化IoC容器所做的主要操作如下:
![AbstractApplicationContext-constructor](https://chentianming11.github.io/images/spring/ioc/AbstractApplicationContext-constructor.png)

从上面的代码可以看出，默认将会使用路径匹配资源解析器`PathMatchingResourcePatternResolver`进行配置资源解析。继续跟踪构造方法源码：
![PathMatchingResourcePatternResolver-constructor](https://chentianming11.github.io/images/spring/ioc/PathMatchingResourcePatternResolver-constructor.png)

在设置完容器的资源解析器之后，接下来`ClassPathXmlApplicationContext`调用其父类 `AbstractRefreshableConfigApplicationContext`的`setConfigLocations()`方法设置配置定位路径，该方法的源码如下:

![AbstractRefreshableConfigApplicationContext-setConfigLocations](https://chentianming11.github.io/images/spring/ioc/AbstractRefreshableConfigApplicationContext-setConfigLocations.png)

**至此，Spring IoC容器已经设置到配置资源解析器，并加载好了配置定位路径，接下来就要正式启动容器了**。

### 开始启动

Spring IoC 容器对`Bean`配置资源的载入是从`refresh()`方法开始的，`refresh()`是一个模板方法，规定了IoC 容器的启动流程，具体逻辑要交给其子类去实现。`refresh()`方法源码如下：
![AbstractApplicationContext-refresh](https://chentianming11.github.io/images/spring/ioc/AbstractApplicationContext-refresh.png)

`refresh()`方法主要为 IoC 容器`Bean`的生命周期管理提供条件，载入配置信息是在第二步`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`进行的。

### 创建容器

`obtainFreshBeanFactory()`方法调用子类容器的`refreshBeanFactory()`方法，启动容器载入`Bean`配置信息，代码如下:
![AbstractApplicationContext-obtainFreshBeanFactory](https://chentianming11.github.io/images/spring/ioc/AbstractApplicationContext-obtainFreshBeanFactory.png)

容器真正调用的是其子类`AbstractRefreshableApplicationContext`实现的`refreshBeanFactory()`方法，方法的源码如下:
![AbstractRefreshableApplicationContext-refreshBeanFactory](https://chentianming11.github.io/images/spring/ioc/AbstractRefreshableApplicationContext-refreshBeanFactory.png)

该方法中最重要的是调用了`loadBeanDefinitions(beanFactory)`方法，从方法名就能推断出，该方法完成了`BeanDefinition`的加载和注册！

### 载入配置路径

`loadBeanDefinitions(beanFactory)`真正调用的是其子类`AbstractXmlApplicationContext`的实现，源码如下：
![AbstractXmlApplicationContext-loadBeanDefinitions1](https://chentianming11.github.io/images/spring/ioc/AbstractXmlApplicationContext-loadBeanDefinitions1.png)

![AbstractXmlApplicationContext-loadBeanDefinitions2](https://chentianming11.github.io/images/spring/ioc/AbstractXmlApplicationContext-loadBeanDefinitions2.png)

由于我们使用`ClassPathXmlApplicationContext`作为例子分析，因此`getConfigResources`的返回值为`null`，因此程序执行`reader.loadBeanDefinitions(configLocations)`。

在`XmlBeanDefinitionReader`的抽象父类`AbstractBeanDefinitionReader`中定义了载入过程。源码如下：
![AbstractXmlApplicationContext-loadBeanDefinitions](https://chentianming11.github.io/images/spring/ioc/AbstractXmlApplicationContext-loadBeanDefinitions.png)

`AbstractRefreshableConfigApplicationContext`的`loadBeanDefinitions(String... locations)`方法实际上是调用 `AbstractBeanDefinitionReader`的`loadBeanDefinitions()`方法。

从对`AbstractBeanDefinitionReader`的`loadBeanDefinitions()`方法源码分析可以看出该方法就做了两件事:

1. 调用资源加载器的获取资源方法`resourceLoader.getResource(location)`，获取到要加载的资源。
2. 真正执行加载功能是其子类`XmlBeanDefinitionReader`的`loadBeanDefinitions()`方法。

### 获取配置资源

`resourceLoader.getResource(location)`真正调用的是`DefaultResourceLoader`实现的方法，源码如下：
![DefaultResourceLoader-getResource](https://chentianming11.github.io/images/spring/ioc/DefaultResourceLoader-getResource.png)

`getResource()`支持了多种途径加载资源，在本例中，最终执行的是从容器自身类路径中资源中获取。即执行`getResourceByPath(location)`方法，最终返回的是`ClassPathContextResource`实例。至此，完成了配置资源的获取。

### 加载配置

在获取配置资源之后，接下来执行`AbstractBeanDefinitionReader`的`loadBeanDefinitions(resource)`方法。源码如下：
![AbstractXmlApplicationContext-loadBeanDefinitions2](https://chentianming11.github.io/images/spring/ioc/AbstractXmlApplicationContext-loadBeanDefinitions2.png)

`loadBeanDefinitions(resource)`真正调用的是子类`XmlBeanDefinitionReader`的方法。
![XmlBeanDefinitionReader-loadBeanDefinitions](https://chentianming11.github.io/images/spring/ioc/XmlBeanDefinitionReader-loadBeanDefinitions.png)

在上面的实现中，先将xml配置资源解析成了一个`Document`对象，然后根据该对象执行注册`BeanDefinition`。注册`BeanDefinition`源码如下：
![XmlBeanDefinitionReader-registerBeanDefinitions](https://chentianming11.github.io/images/spring/ioc/XmlBeanDefinitionReader-registerBeanDefinitions.png)

这里进一步调用了`DefaultBeanDefinitionDocumentReader`的`registerBeanDefinitions()`方法。
![DefaultBeanDefinitionDocumentReader-registerBeanDefinitions](https://chentianming11.github.io/images/spring/ioc/DefaultBeanDefinitionDocumentReader-registerBeanDefinitions.png)

![DefaultBeanDefinitionDocumentReader-parseBeanDefinitions](https://chentianming11.github.io/images/spring/ioc/DefaultBeanDefinitionDocumentReader-parseBeanDefinitions.png)

从源码可以看出，加载xml配置时，如果碰到`<import>`，会先导入该标签引用的资源；如果碰到`<alias>`，会将定义的别名注册到容器中；如果是`<Bean>`，则会解析并注册`BeanDefinition`，这也是我们关注的重点。
![DefaultBeanDefinitionDocumentReader-processBeanDefinition](https://chentianming11.github.io/images/spring/ioc/DefaultBeanDefinitionDocumentReader-processBeanDefinition.png)

在`delegate.parseBeanDefinitionElement(ele)`方法中完成了`BeanDefinition`的解析处理，具体解析逻辑不是本文的重点，这里只需要知道，此时`BeanDefinitionHolder`中包含了xml配置文件中的各种信息。解析完成之后，通过`BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry())`进行注册。

### 向容器注册

`BeanDefinitionReaderUtils`的`registerBeanDefinition()`方法源码如下：
![BeanDefinitionReaderUtils-registerBeanDefinition](https://chentianming11.github.io/images/spring/ioc/BeanDefinitionReaderUtils-registerBeanDefinition.png)

实际的注册逻辑调用了`BeanDefinitionRegistry`类的`registerBeanDefinition()`方法。而`BeanDefinitionRegistry`的实现类为`DefaultListableBeanFactory`，继续跟踪源码：
![DefaultListableBeanFactory-registerBeanDefinition](https://chentianming11.github.io/images/spring/ioc/DefaultListableBeanFactory-registerBeanDefinition.png)

从这里可以发现，`BeanDefinition`信息实际注册到了`DefaultListableBeanFactory`的`beanDefinitionMap`属性中。这里请注意，`DefaultListableBeanFactory`是一个`BeanFactory`。通过源码分析发现，不管是`ClassPathXmlApplicationContext`还是`AnnotationConfigApplicationContext`都持有`DefaultListableBeanFactory`实例。也就是说，`BeanDefinition`已经注册到容器中了，通过容器，可以获取所有的`BeanDefinition`注册信息，为后续的`Bean`的创建和依赖注入提供了基础！


## 基于Annotation的配置资源加载和注册

目前更常见的的是通过注解来配置`Bean`，比如在类上加`@Controller`、`@Service`、`@Component`等。下面我们就来分析基于Annotation的配置资源加载和注册过程。

### 寻找入口

基于Annotation的Spring IoC容器实例是`AnnotationConfigApplicationContext`，我们可以通过它的构造方法进行分析。最关键的两个构造方法如下：

```java
// 创建一个新的 AnnotationConfigApplicationContext，从给定的带注解的加载bean定义并自动刷新上下文。
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();
}

// 创建一个新的 AnnotationConfigApplicationContext，扫描给定包中的 bean 定义并自动刷新上下文。
public AnnotationConfigApplicationContext(String... basePackages) {
        this();
        scan(basePackages);
        refresh();
        }
```

注解驱动的Spring容器有两种创建方式：

1. 直接传入带有相关注解的类。
2. 指定要扫描的基础包，将包下面所有带相关注解的Bean全部加载进去。

接下来，分别介绍这两种容器创建方式。

### 直接传入带有相关注解的类

显然，`BeanDefinition`注册的关键代码在`register(annotatedClasses)`中，我们直接看相关源码：
![AnnotationConfigApplicationContext-register](https://chentianming11.github.io/images/spring/ioc/AnnotationConfigApplicationContext-register.png)

这里调用了`AnnotatedBeanDefinitionReader`的`register()`方法，继续跟进去：
![AnnotatedBeanDefinitionReader-register](https://chentianming11.github.io/images/spring/ioc/AnnotatedBeanDefinitionReader-register.png)

![AnnotatedBeanDefinitionReader-doRegisterBean](https://chentianming11.github.io/images/spring/ioc/AnnotatedBeanDefinitionReader-doRegisterBean.png)

从上面的源码我们可以看出，注册注解`Bean`定义类的基本步骤如下：

1. 使用注解元数据解析器解析注解`Bean`中关于作用域的配置。
2. 使用`AnnotationConfigUtils`的`processCommonDefinitionAnnotations()`方法处理注解`Bean`定义类中通用的注解。
3. 使用`AnnotationConfigUtils`的`applyScopedProxyMode()`方法创建对于作用域的代理对象。
4. 通过`BeanDefinitionReaderUtils`向容器注册`Beandefinition`。

下面，我们详细分析这4个步骤：

#### 使用注解元数据解析器解析注解`Bean`中关于作用域的配置

通过调用`AnnotationScopeMetadataResolver`类的`resolveScopeMetadata()`方法解析注解`Bean`定义类的作用域元信息。
![AnnotationScopeMetadataResolver-resolveScopeMetadata](https://chentianming11.github.io/images/spring/ioc/AnnotationScopeMetadataResolver-resolveScopeMetadata.png)

#### 使用`AnnotationConfigUtils`的`processCommonDefinitionAnnotations()`方法处理注解`Bean`定义类中通用的注解

![AnnotationConfigUtils-processCommonDefinitionAnnotations](https://chentianming11.github.io/images/spring/ioc/AnnotationConfigUtils-processCommonDefinitionAnnotations.png)

显然，这里在解析`@Lazy`、`@Primary`、`@DependsOn`等注解值，然后给`BeanDefinition`对应的字段赋值。

#### 使用`AnnotationConfigUtils`的`applyScopedProxyMode()`方法创建对于作用域的代理对象

![AnnotationConfigUtils-applyScopedProxyMode](https://chentianming11.github.io/images/spring/ioc/AnnotationConfigUtils-applyScopedProxyMode.png)

该方法根据作用域`@Scope`注解的值，为`Bean`定义应用相应的代理模式，主要是在Spring面向切面编程(AOP)中使用。

#### 通过`BeanDefinitionReaderUtils`向容器注册`Beandefinition`

该方法就是将`Beandefinition`注册到容器中，前面详细介绍过，不再赘述。

### 指定要扫描的基础包

当创建注解处理容器时，如果传入的初始参数是注解`Bean`定义类所在的包时，注解容器将扫描给定的包及其子包，将扫描到的注解`Bean`定义载入并注册。

#### ClassPathBeanDefinitionScanner 扫描给定的包及其子包

`AnnotationConfigApplicationContext`的`scan()`方法实际调用了`ClassPathBeanDefinitionScanner`的`scan()`方法，源码如下：

![ClassPathBeanDefinitionScanner-scan](https://chentianming11.github.io/images/spring/ioc/ClassPathBeanDefinitionScanner-scan.png)

![ClassPathBeanDefinitionScanner-doScan](https://chentianming11.github.io/images/spring/ioc/ClassPathBeanDefinitionScanner-doScan.png)

从源码可以看到，在`doScan()`方法里面，扫描了基础包下所有包含`Bean`定义注解的类，并且给`BeanDefinition`设置了对应的属性值。

#### 向容器注册

在`doScan()`方法最后，调用了`registerBeanDefinition(definitionHolder, this.registry)`向容器注册。这个在前面介绍过，不再赘述。

## 小结

不管是基于`xml`还是注解方式，都是将配置信息解析到`BeanDefinition`中，然后再注册到容器中。至此，在容器中就能获取所有`Bean`定义信息，接下来就是`Bean`实例的创建和依赖注入(DI)，下一章节再见！

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)

