---
title: 一文搞懂Spring IoC实现原理
tags: spring
categories: spring
abbrlink: 800005253
date: 2021-06-07 08:17:52
---

Spring是日常工作中最常用的框架，IoC又是Spring的核心实现之一。**IoC(Inversion of Control)的意思是控制反转，就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现**。在Spring中，常见的包括**web IoC容器**、**xml IoC容器**和**Annotation IoC容器**等， 本文详细介绍了这些IoC容器的初始化过程。

<!--more-->


## Web IOC容器初始化

对于Spring-MVC来说，最核心的类是`DispatcherServlet`，它负责将请求转发给各个`Controller`进行处理。类图如下：
![DispatcherServlet](https://chentianming11.github.io/images/spring/ioc/DispatcherServlet.png)

通过类图可以发现，`DispatcherServlet`实际上最终继承了`HttpServlet`，而`HttpServlet`是web容器启动的核心类，它有一个`init()`方法来做一些初始化操作。在`HttpServletBean`重写了`init()`，具体代码如下：
![HttpServletBean-init](https://chentianming11.github.io/images/spring/ioc/HttpServletBean-init.png)

可以看到，在`init()`方法中，真正完成初始化容器动作的逻辑其实在`initServletBean()`方法中，而该方法的具体实现在其子类`FrameworkServlet`中。
![FrameworkServlet-initServletBean](https://chentianming11.github.io/images/spring/ioc/FrameworkServlet-initServletBean.png)

从上面的代码可以明显看出，真正初始化web IoC容器是在`initWebApplicationContext()`方法中实现的。具体代码如下：
![FrameworkServlet-initWebApplicationContext](https://chentianming11.github.io/images/spring/ioc/FrameworkServlet-initWebApplicationContext.png)

![FrameworkServlet-configureAndRefreshWebApplicationContext](https://chentianming11.github.io/images/spring/ioc/FrameworkServlet-configureAndRefreshWebApplicationContext.png)

在这个方法中，建立了父子容器的关联关系，并最终调用了`configureAndRefreshWebApplicationContext(cwac)`方法来初始化IoC容器。方法最终调用了`AbstractApplicationContext`的`refresh()`方法。

> 父容器（Root WebApplicationContext）：对三层架构中的service层、dao层进行配置，如业务bean，数据源(DataSource)等。通常情况下，配置文件的名称为applicationContext.xml。在web应用中，其一般通过ContextLoaderListener来加载。

> 子容器（Servlet WebApplicationContext）：对三层架构中的web层进行配置，如控制器(controller)、视图解析器(view resolvers)等相关的bean。通过spring mvc中提供的DispatchServlet来加载配置，通常情况下，配置文件的名称为spring-servlet.xml。

> 更多父子容器介绍可参考[spring和springmvc父子容器概念介绍](https://www.cnblogs.com/grasp/p/11042580.html)。

![AbstractApplicationContext-refresh](https://chentianming11.github.io/images/spring/ioc/AbstractApplicationContext-refresh.png)

![AbstractApplicationContext-refresh2](https://chentianming11.github.io/images/spring/ioc/AbstractApplicationContext-refresh2.png)

`refresh()`是真正启动IoC容器的入口，后续会详细介绍。IoC容器初始化以后，最后调用了 `DispatcherServlet`的`onRefresh()`方法，在`onRefresh()`方法中又是直接调用 `initStrategies()`方法初始化 SpringMVC 的九大组件。
![DispatcherServlet-onRefresh](https://chentianming11.github.io/images/spring/ioc/DispatcherServlet-onRefresh.png)

## 基于Xml的IoC容器的初始化


