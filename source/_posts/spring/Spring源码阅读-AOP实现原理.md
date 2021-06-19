---
title: 【Spring源码阅读】AOP实现原理
date: 2021-06-19 14:18:01
tags: spring
categories: spring
---

AOP是Aspect Oriented Programming的缩写，意思是面向切面编程。可以通过预编译方式或者运行时动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。本文会从源码的角度详细阐述Spring中AOP的实现原理。

<!--more-->

## AOP中几个重要概念

### 通知（Advice）

在AOP术语中，切面的工作被称为通知。通知定义了切面是什么以及何时使用。spring切面可以应用五种类型的通知：

- 前置通知（Before）：在目标方法调用之前调用通知功能。
- 后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么。
- 返回通知（After-returning）：在目标方法成功执行之后调用通知。
- 异常通知（After-throwing）：在目标方法抛出异常后调用通知。
- 环绕通知（Around）：通知包裹了被通知方法，在被通知方法调用之前和之后执行自定义行为。

### 连接点（join point）

连接点是应用程序执行过程中能插入切面的一个点。这个点可以是调用方法时，抛出异常 时，甚至是修改一个字段时。切面代码可以利用这些点插入到应用的正常流程中，并添加新的行为。

### 切点（pointcut）

如果说通知定义了切面的“什么”和“何时”的话，那么切点就定义了“何处”。切点的定义会匹配通知所要织入的一个或多个连接点。我们通常使用明确的类和方法名或者正则表达式定义所匹配的类和方法名来指定这些切点。

##### 切面（Aspect）

切面是通知和切点的结合。通知和切点共同定义了切面的全部内容--它是什么，在何时何处完成其功能。

##### 引入（Introduction）

引入允许我们向现有类添加新方法或属性。

##### 织入（Weaving）

织入是把切面应用到目标对象并创建新的代理对象的过程。切面在指定连接点被织入到目标对象中。在目标对象的生命周期里，有多个点可以进行织入。

- 编译期：切面在目标类编译时被织入。AspectJ的织入编译器就是以这种方式织入切面的。
- 类加载期：切面在目标类加载到JVM时被织入。这种方式需要特使的类加载器，它可以在目标类被引入应用之前增强该目标类的字节码。AspectJ 5的加载时织入就支持这种方式织入切面。
- 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态创建一个代理对象。String AOP就是以这种方式织入切面。

## Spring AOP 源码分析

### 寻找入口

通过上一章节的分析，我们知道，`Bean`在完成依赖注入之后，会进行初始化操作，然后整个容器才算启动完成。而此时，我们调用的`Bean`的方法，如果上面有配置有AOP相关注解的话，那么AOP代码是会织入进去的。这说明，此时`Bean`实例已经是一个代理对象了。那么`Bean`实例是什么时候被代理的呢，显然应该是在初始化阶段。我们继续来看一下初始化的源代码：
![AbstractAutowireCapableBeanFactory-initializeBean.png](https://chentianming11.github.io/images/spring/aop/AbstractAutowireCapableBeanFactory-initializeBean.png)

可以看到，在`wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName)`以及`wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);`执行之后，都返回了包装之后的实例`wrappedBean`。而这两个方法底层分别调用的是`BeanPostProcessor`的`postProcessBeforeInitialization()`和`postProcessAfterInitialization()`方法。**因此，我们可以推断出AOP一定是通过某个`BeanPostProcessor`来实现的**。

通过代码查找名称中带有proxy的`BeanPostProcessor`实现类，其继承关系如下：
![BeanPostProcessor-proxy](https://chentianming11.github.io/images/spring/aop/BeanPostProcessor-proxy.png)

可以很明显的看出，所有Proxy类都源自一个抽象父类`AbstractAutoProxyCreator`，该类重写了`postProcessAfterInitialization()`，这就是AOP的入口。

### 选择代理策略

进入`postProcessAfterInitialization()`方法，我们发现调到了一个非常核心的方法`wrapIfNecessary()`， 其源码如下:
![AbstractAutoProxyCreator-wrapIfNecessary](https://chentianming11.github.io/images/spring/aop/AbstractAutoProxyCreator-wrapIfNecessary.png)

![AbstractAutoProxyCreator-createProxy](https://chentianming11.github.io/images/spring/aop/AbstractAutoProxyCreator-createProxy.png)

整个过程跟下来，我发现最终调用的是`proxyFactory`的`getProxy()`方法。从名字就能看出来，`proxyFactory`是管理各种`Proxy`的工厂。继续跟进发现它调用了子类`ProxyCreatorSupport`的`createAopProxy()`来获取最终代理对象的。
![ProxyFactory-getProxy](https://chentianming11.github.io/images/spring/aop/ProxyFactory-getProxy.png)

![ProxyCreatorSupport-createAopProxy](https://chentianming11.github.io/images/spring/aop/ProxyCreatorSupport-createAopProxy.png)

继续跟进，发现最终调用的是`DefaultAopProxyFactory`的`createAopProxy`，这里根据不同情况分别返回了`JdkDynamicAopProxy`和`ObjenesisCglibAopProxy`代理。即使用JDK或者CGLIB进行动态代理。

### 调用代理方法

上面我们已经了解到Spring提供了两种方式来生成代理，分别是`JDKProxy 和 CGLib。下面我们来研究一下Spring如何使用JDK来生成代理对象，具体的生成代码放在`JdkDynamicAopProxy`这个类中， 直接上相关代码:
![JdkDynamicAopProxy-getProxy](https://chentianming11.github.io/images/spring/aop/JdkDynamicAopProxy-getProxy.png)

`InvocationHandler`是JDK动态代理的核心，生成的代理对象的方法调用都会委托到`InvocationHandler.invoke()`方法。而从`JdkDynamicAopProxy`的源码我们可以看到这个类其实也实现了`InvocationHandle`，我们直接上源码看`invoke()`方法:
![JdkDynamicAopProxy-invoke](https://chentianming11.github.io/images/spring/aop/JdkDynamicAopProxy-invoke.png)

上面方法的实现思路非常清晰：

1. 首先获取应用到此方法上的通知链(Interceptor Chain)。
2. 如果有通知，则应用通知，并执行`JoinPoint`; 如果没有通知，则直接反射执行。

这里的关键是**通知链是如何获取的以及它又是如何执行的**，我们接下来进行逐一分析。

#### 通知链是如何获取的

从上面的代码可以看到，通知链是通过`Advised.getInterceptorsAndDynamicInterceptionAdvice()`这个方法来获取的，我们来看下这个方法的实现逻辑:
![AdvisedSupport-getInterceptorsAndDynamicInterceptionAdvice](https://chentianming11.github.io/images/spring/aop/AdvisedSupport-getInterceptorsAndDynamicInterceptionAdvice.png)

![DefaultAdvisorChainFactory-getInterceptorsAndDynamicInterceptionAdvice](https://chentianming11.github.io/images/spring/aop/DefaultAdvisorChainFactory-getInterceptorsAndDynamicInterceptionAdvice.png)

从提供的配置实例`config`中获取`advisor`列表，遍历处理这些`advisor`。如果是`IntroductionAdvisor`，则判断此 `Advisor`能否应用到目标类`targetClass`上.如果是`PointcutAdvisor`,则判断此`Advisor`能否应用到目标方法 `Method`上.将满足条件的`Advisor`通过`AdvisorAdaptor`转化成`Interceptor`列表返回。

#### 通知链如何执行的

上面方法执行完成后，`Advised`中配置能够应用到连接点(JoinPoint)或者目标类(Target Object) 的`Advisor`全部被转化成了`MethodInterceptor`，接下来我们再看下得到的拦截器链是怎么起作用的。实际调用了`ReflectiveMethodInvocation`
的`proceed`执行。
![DReflectiveMethodInvocation-proceed](https://chentianming11.github.io/images/spring/aop/ReflectiveMethodInvocation-proceed.png)

显然，上面的方法主要目的是实现链式调用，而具体拦截器里面具体执行了什么，是如何触发通知的，还是得回到拦截器链相关的代码里面。

### 触发通知

我们前面讲过，拦截器链式通过`DefaultAdvisorChainFactory`的`getInterceptorsAndDynamicInterceptionAdvice()`方法生成的。在该方法中，有一个适配器和注册过程，通过配置Spring预先设计好拦截器，Spring加入了它对AOP实现的处理。
![DefaultAdvisorChainFactory-getInterceptorsAndDynamicInterceptionAdvice2](https://chentianming11.github.io/images/spring/aop/DefaultAdvisorChainFactory-getInterceptorsAndDynamicInterceptionAdvice2.png)

所有的拦截器都是调用了`DefaultAdvisorAdapterRegistry`的`getInterceptors()`获得的。在`DefaultAdvisorAdapterRegistry`的构造方法中，提前注册好了各种通知适配器。正是这些适配器的实现，为 Spring AOP提供了编织能力。下面以`MethodBeforeAdviceAdapter`为例，看具体的实现:
![DefaultAdvisorAdapterRegistry](https://chentianming11.github.io/images/spring/aop/DefaultAdvisorAdapterRegistry.png)

![MethodBeforeAdviceAdapter](https://chentianming11.github.io/images/spring/aop/MethodBeforeAdviceAdapter.png)

Spring AOP为了实现`advice`的织入，设计了特定的拦截器对这些功能进行了封装。比如上述例子使用的就是`MethodBeforeAdviceInterceptor`，源码如下：
![MethodBeforeAdviceInterceptor](https://chentianming11.github.io/images/spring/aop/MethodBeforeAdviceInterceptor.png)

可以看到，对于`MethodBeforeAdviceInterceptor`，在执行该拦截器的时候会先调用通知的`before()`方法，然后继续执行后面的拦截器，其它拦截器的原理与此类似。通过这种方式，就能将应用到该方法的所有通知都执行一遍了。


> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)
