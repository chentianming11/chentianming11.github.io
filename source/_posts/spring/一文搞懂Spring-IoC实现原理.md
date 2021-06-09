---
title: 一文搞懂Spring IoC实现原理
tags: spring
categories: spring
abbrlink: 800005253
date: 2021-06-07 08:17:52
---

Spring是日常工作中最常用的框架，IoC又是Spring的核心实现之一。**IoC(Inversion of Control)的意思是控制反转，就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现**。在Spring中，主要包括**web IoC容器**、**xml IoC容器**和**Annotation IoC容器**等， 本文详细介绍了这些IoC容器的初始化过程。

<!--more-->

## Spring核心容器类

为了后续能够了解IoC容器的初始化过程，我们先得了解三个Spring核心容器类，分别是`BeanFactory`、`BeanDefinition`和`BeanDefinitionReader`。

### BeanFactory

**`BeanFactory`是一个接口，它是Spring中工厂的顶层规范，是Spring Ioc容器的核心接口，它定义了`getBean()`、`containsBean()`等管理Bean的通用方法**。Spring的容器都是它的具体实现，比如`DefaultListableBeanFactory`、`ApplicationContext`等等。源码如下：

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

在`BeanFactory`里只对IoC容器的基本行为作了定义，根本不关心你的`Bean`是如何定义怎样加载的。 正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。


而要知道工厂是如何产生对象的，我们需要看具体的IoC容器实现，Spring提供了许多IoC容器的实现。比如`WebApplicationContext`、`ClassPathXmlApplicationContext`、`AnnotationConfigApplicationContext`等等，它们都实现了`ApplicationContext`接口。`ApplicationContext`可以看作是Spring提供的一个高级的IoC容器，它除了能够提供IoC容器的基本功能外，还为用户提供了各种附加服务，比如访问资源(`ResourcePatternResolver`)、支持应用事件(`ApplicationEventPublisher`)等等。

### BeanDefinition

Spring IOC容器管理了定义的各种`Bean`对象及其相互的关系，`Bean`对象在Spring实现中是以`BeanDefinition`来描述的。类图如下：

![BeanDefinition](https://chentianming11.github.io/images/spring/ioc/BeanDefinition.png)

### BeanDefinitionReader

`Bean`的解析过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。`Bean`的解析主要就是对Spring配置文件的解析。解析过程主要通过`BeanDefinitionReader` 来完成，类图如下：

![BeanDefinitionReader](https://chentianming11.github.io/images/spring/ioc/BeanDefinitionReader.png)

## Web IOC容器初始化

对于Spring-MVC来说，最核心的类是`DispatcherServlet`，它负责将请求转发给各个`Controller`进行处理。类图如下：
![DispatcherServlet](https://chentianming11.github.io/images/spring/ioc/DispatcherServlet.png)

通过类图可以发现，`DispatcherServlet`实际上最终继承了`HttpServlet`，而`HttpServlet`是web容器启动的核心类，它有一个`init()`方法来做一些初始化操作。在`HttpServletBean`重写了`init()`，具体代码如下：
![HttpServletBean-init](https://chentianming11.github.io/images/spring/ioc/HttpServletBean-init.png)

可以看到，在`init()`方法中，真正完成初始化容器动作的逻辑其实在`initServletBean()`方法中，而该方法的具体实现在其子类`FrameworkServlet`中。
![FrameworkServlet-initServletBean](https://chentianming11.github.io/images/spring/ioc/FrameworkServlet-initServletBean.png)

从上面的代码可以明显看出，真正初始化web IoC容器是在`initWebApplicationContext()`方法中实现的。具体代码如下：
![FrameworkServlet-initWebApplicationContext](https://chentianming11.github.io/images/spring/ioc/FrameworkServlet-initWebApplicationContext.png)















> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)


