---
title: 【Spring源码阅读】MVC实现原理
tags: spring
categories: spring
abbrlink: 3374948667
date: 2021-07-03 11:03:52
---

Spring MVC是当前web应用使用最为广泛的框架，本文将会从源码角度详细分析Spring MVC实现原理。

<!--more-->

## Spring MVC基本流程

在具体讲解之前，我们先从宏观上简单介绍Spring MVC的处理流程，了解用户从发起请求到获得响应的完整流程。

![mvc-all](https://chentianming11.github.io/images/spring/mvc/mvc-all.png)

流程简介：

1. 用户发起请求到`DispatcherServlet`（前端控制器）
2. `DispatcherServlet`通过`HandlerMapping`（处理映射器）寻找用户要请求的`Handler`
3. `HandlerMapping`返回执行链，包含两部分内容：
    1. 处理器对象：`Handler`
    2. `HandlerInterceptor`（拦截器）的集合

4. 前端控制器通过`HandlerAdapter`（处理器适配器）对`Handler`进行适配包装
5. 调用包装后的`Handler`中的方法，处理业务
6. 处理业务完成，返回`ModelAndView`对象，包含两部分
    1. `Model`：模型数据
    2. `View`：视图名称，不是真正的视图
    
7. `DispatcherServlet`获取处理得到的`ModelAndView`对象
8. `DispatcherServlet`将视图名称交给`ViewResolver`（视图解析器），查找视图
9. `ViewResolver`返回真正的视图对象给`DispatcherServlet`
10. `DispatcherServlet`把`Model`（数据模型）交给视图对象进行渲染
11. 返回渲染后的视图
12. 将最终的视图返回用户，产生响应

## Spring MVC九大组件

### HandlerMapping

`HandlerMapping`是用来查找`Handler`的，也就是处理器，具体表现形式可以是类也 可以是方法。比如，标注了`@RequestMapping`的每个`method`都可以看成是一个`Handler`，由`Handler`来负责实际的请求处理。`HandlerMapping`在请求到达之后， 它的作用便是找到请求相应的处理器`Handler`和`Interceptors`。

### HandlerAdapter

从名字上看，这是一个适配器。因为Spring MVC中`Handler`可以是任意形式的，只要能够处理请求就行。但是把请求交给 `Servlet`的时候，由于`Servlet`的方法结构都是如`doService(HttpServletRequest req, HttpServletResponse resp)`这样的形式，让固定的`Servlet`处理方法调用`Handler`来进行处理，这一步工作便是`HandlerAdapter`要做的事。

### HandlerExceptionResolver

从这个组件的名字上看，这个就是用来处理`Handler`过程中产生的异常情况的组件。 具体来说，此组件的作用是根据异常设置 `ModelAndView`, 之后再交给`render()`方法进行渲染，而`render()`便将`ModelAndView`渲染成页面。 不过有一点，`HandlerExceptionResolver`只是用于解析对请求做处理阶段产生的异常，而渲染阶段的异常则不归他管了，这也是Spring MVC组件设计的一大原则（分工明确互不干涉）。

### ViewResolver

视图解析器，相信大家对这个应该都很熟悉了。因为通常在Spring MVC的配置文件中， 都会配上一个该接口的实现类来进行视图的解析。 这个组件的主要作用，便是将`String`类型的视图名和`Locale`解析为`View`类型的视图。这个接口只有一个 `resolveViewName()`方法。从方法的定义就可以看出，`Controller`层返回的`String`类型的视图名`viewName`， 最终会在这里被解析成为`View`。`View`是用来渲染页面的，也就是说，它会将程序返回的参数和数据填入模板中，最终生成`html `文件。`ViewResolver`在这个过程中，主要做两件大事，即`ViewResolver`会找到渲染所用的模板(使用什么模板来渲染?)和所用 的技术(其实也就是视图的类型，如 JSP等)填入参数。默认情 况下，Spring MVC 会为我们自动配置一个 `InternalResourceViewResolver`，这个是针 对 JSP 类型视图的。

### RequestToViewNameTranslator

这个组件的作用，在于从`Request`中获取`viewName`. 因为`ViewResolver`是根据`ViewName`查找`View`, 但有的 `Handler`处理完成之后，没有设置`View`也没有设置`ViewName`， 便要通过这个组件来从`Request`中查找`viewName`。

### LocaleResolver

在上面我们有看到`ViewResolver`的`resolveViewName()`方法，需要两个参数。那么第二个参数`Locale`是从哪来的呢，这就是`LocaleResolver`要做的事了。 `LocaleResolver`用于从`request`中解析出`Locale`, 在中国大陆地区，`Locale`当然就会是`zh-CN`之类， 用来表示一个区域。这个类也是`i18n`的基础。

### ThemeResolver

从名字便可看出，这个类是用来解析主题的。主题就是样式，图片以及它们所形成的显示效果的集合。

### MultipartResolver

其实这是一个大家很熟悉的组件，`MultipartResolver`用于处理上传请求，通过将普通的`Request`包装成 `MultipartHttpServletRequest`来实现。`MultipartHttpServletRequest`可以通过`getFile()`直接获得文件，如果是多个文件上传，还可以通过调用`getFileMap`得到`Map`这样的结构。`MultipartResolver`的作用就是用来封装普通 的`request`，使其拥有处理文件上传的功能。

### FlashMapManager

`FlashMap`用于重定向`Redirect`时的参数数据传递, 只需要在`redirect`之前，将要传递的数据写入`request`(可以通过 `ServletRequestAttributes.getRequest()` 获得) 的属性`OUTPUT_FLASH_MAP_ATTRIBUTE`中，这样在 `redirect`之后的`handler`中，Spring就会自动将其设置到`Model`中，后续就可以直接从`Model`中取得数据了。而 `FlashMapManager`就是用来管理`FlashMap`的。


## Spring MVC源码分析

### 初始化阶段

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

IoC容器初始化以后，最后调用了 `DispatcherServlet`的`onRefresh()`方法，在`onRefresh()`方法中又是直接调用 `initStrategies()`方法初始化 SpringMVC 的九大组件。
![DispatcherServlet-onRefresh](https://chentianming11.github.io/images/spring/ioc/DispatcherServlet-onRefresh.png)

### 运行调用阶段

当前置的初始化完成之后，Spring MVC就可以接受并处理请求了。下面，我们从发起一个请求为例，详细介绍其执行过程。对于Spring MVC而言，所有请求都会经过`DispatcherServlet`的`doService()`方法。而`doService()`中的核心逻辑由 `doDispatch()`实现，源代码如下:
![DispatcherServlet-doDispatch](https://chentianming11.github.io/images/spring/mvc/DispatcherServlet-doDispatch.png)

从源码可以看出，请求处理的核心步骤如下：

1. 确定当前请求的处理器
2. 获取当前请求的处理适配器
3. 拦截器前置处理
4. 真正调用处理器
5. 拦截器后置处理

下面我们一个一个步骤进行分析。

####  确定当前请求的处理器

`getHandler(HttpServletRequest request)`源码如下：
![DispatcherServlet-getHandler](https://chentianming11.github.io/images/spring/mvc/DispatcherServlet-getHandler.png)

从源码可以看出，直接遍历`handlerMappings`，第一个匹配中的`HandlerMapping`对象，再从`HandlerMapping`对象获取处理器。因此问题的关键就是`handlerMappings`有哪些？它们在实例化的时候又做了哪些准备。

我们知道，在初始化阶段会初始化九大组件，其中就有`HandlerMapping`。也就是在这个时刻，`handlerMappings`准备好了。`HandlerMapping`是一个接口，它有很多个实现，但在实际应用中，我们更常见的是`RequestMappingHandlerMapping`。下面我们就以`RequestMappingHandlerMapping`为例进行分析，其类图如下：
![RequestMappingHandlerMapping](https://chentianming11.github.io/images/spring/mvc/RequestMappingHandlerMapping.png)

`RequestMappingHandlerMapping`本身也是Spring容器的一个`Bean`实例。从类图中可以看到，其抽象父类实现了`AbstractHandlerMethodMapping`实现了`InitializingBean`接口，因此可以肯定在它的`afterPropertiesSet()`实现中做了一些初始化处理。
![AbstractHandlerMethodMapping-initHandlerMethods](https://chentianming11.github.io/images/spring/mvc/AbstractHandlerMethodMapping-initHandlerMethods.png)

在`detectHandlerMethods(beanName)`方法中实现了检测并注册处理器方法的方逻辑。在这里，**建立了`url`和`handler`人以及`method`的映射关系**。

继续回到`HandlerMapping`，在匹配到`HandlerMapping`，调用了它的`getHandler()`方法，具体实现在它的抽象父类`AbstractHandlerMapping`中，源码如下：
![AbstractHandlerMapping-getHandler](https://chentianming11.github.io/images/spring/mvc/AbstractHandlerMapping-getHandler.png)

可以看到，最关键的步骤有2个：

1. 第一步是根据`request`查找出具体的`Handler`（这里实际上是方法法处理器），这个很好理解。简单来说就是根据url查找前面注册好的`Method`和`Handler`(这里是指`Controller`实例)，并包装成`HandlerMethod`返回。
2. 第二步就是根据`request`查找对应的拦截器实例，并将处理器、拦截器包装成处理器执行链`HandlerExecutionChain`返回。
   
![AbstractHandlerMapping-getHandlerExecutionChain](https://chentianming11.github.io/images/spring/mvc/AbstractHandlerMapping-getHandlerExecutionChain.png)

**至此，已经确定了当前请求的处理器，它包含了方法处理器以及拦截器链**。

#### 获取当前请求的处理适配器

获取处理`request`的处理器适配器`handler adapter`。对于本例来说获取的是`RequestMappingHandlerAdapter`，不做过多赘述。

#### 拦截器前置处理

执行拦截器`HandlerInterceptor`的前置处理方法`preHandle()`。

#### 真正调用处理器

执行调用适配器`AbstractHandlerMethodAdapter`的`handle()`方法，进行真正的调用逻辑处理，该方法直接调用了子类的`handleInternal()`方法。
![RequestMappingHandlerAdapter-handleInternal](https://chentianming11.github.io/images/spring/mvc/RequestMappingHandlerAdapter-handleInternal.png)

可以看到，最关键的是调用了`invokeHandlerMethod()`方法。
![RequestMappingHandlerAdapter-invokeHandlerMethod](https://chentianming11.github.io/images/spring/mvc/RequestMappingHandlerAdapter-invokeHandlerMethod.png)

在`invocableMethod.invokeAndHandle()`中完成了`Request`中的参数和方法参数上数据的绑定。Spring MVC中提供两种`Request`参数到方法中参数的绑定方式:通过注解`@RequestParam`进行绑定、通过参数名称进行绑定。

使用注解进行绑定，我们只要在方法参数前面声明`@RequestParam("name")`，就可以 将`request`中参数`name`的值绑定到方法的该参数上。**使用参数名称进行绑定的前提是必须要获取方法中参数的名称**。Spring基于ASM技术实现了运行时获取参数的实现。

![ServletInvocableHandlerMethod-invokeAndHandle](https://chentianming11.github.io/images/spring/mvc/ServletInvocableHandlerMethod-invokeAndHandle.png)

到这里,方法的参数值列表也获取到了,方法反射调用也结束了，并且方法返回值也拿到了。后面就需要对返回结果进行处理了，继续跟进`handleReturnValue()`方法，对于常见的`json`响应体返回，实际调用了`RequestResponseBodyMethodProcessor`的实现方法：
![RequestResponseBodyMethodProcessor-handleReturnValue](https://chentianming11.github.io/images/spring/mvc/RequestResponseBodyMethodProcessor-handleReturnValue.png)

这里根据实际的消息转换器，将方法返回值写入到response中，在执行完后续操作之后，直接将结果输出。

#### 拦截器后置处理

执行拦截器`HandlerInterceptor`的后置处理方法`postHandle()`。

## 总结

总的来说，Spring MVC做的事情还是比较清晰的，在初始化阶段，主要会建立请求url与执行实例和执行方法的映射关系。在调用阶段，根据前面的映射关系，找到对应的方法执行，并将返回结果写入`response`中。当然，在这个过程中，为了解决各种复杂性的问题，引入了非常的设计来应对。但我们只要理解其核心实现即可，过于细节的内容可以先不管。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)