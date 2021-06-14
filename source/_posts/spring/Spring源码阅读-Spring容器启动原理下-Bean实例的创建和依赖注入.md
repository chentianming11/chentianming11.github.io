---
title: 【Spring源码阅读】Spring容器启动原理(下)-Bean实例的创建和依赖注入
date: 2021-06-14 10:22:46
tags: spring
categories: spring
---

在上一章节我们介绍了配置资源的加载和注册，此时，Spring容器已经管理了所有的`Bean`定义相关数据。接下来，就是`Bean`实例的创建和依赖注入(DI)了。**DI(Dependency Injection)依赖注入是指对象是被动接受依赖类而不是自己主动去找，换句话说就是指对象不是从容器中查找它依赖的类，而是在容器实例化对象的时候主动将它依赖的类注入给它**。

<!--more-->

## 寻找入口

`Bean`实例的创建和依赖注入在以下两种情况下触发：

1. 用户第一次调用`getBean()`方法时，IoC容器触发依赖注入。
2. 当`Bean`的懒加载属性为`false`时，在容器初始化时自动触发依赖注入。此时还是会调用到`getBean()`方法。

因此，`Bean`实例的创建和依赖注入的入口就在`getBean()`方法中。在上一章节，我们讲过不管是`ClassPathXmlApplicationContext`还是`AnnotationConfigApplicationContext`都持有`DefaultListableBeanFactory`实例。而它们的`getBean()`就是直接调用了`BeanFactory`的`getBean()`方法。`DefaultListableBeanFactory`的`getBean()`才是真正的入口。`getBean(String)`的实现在其抽象父类`AbstractBeanFactory`中，源码如下：
![AbstractBeanFactory-getBean](https://chentianming11.github.io/images/spring/di/AbstractBeanFactory-getBean.png)

实际调用的是`doGetBean()`方法：
![AbstractBeanFactory-doGetBean1](https://chentianming11.github.io/images/spring/di/AbstractBeanFactory-doGetBean1.png)

![AbstractBeanFactory-doGetBean2](https://chentianming11.github.io/images/spring/di/AbstractBeanFactory-doGetBean2.png)

![AbstractBeanFactory-doGetBean3](https://chentianming11.github.io/images/spring/di/AbstractBeanFactory-doGetBean3.png)

![AbstractBeanFactory-doGetBean4](https://chentianming11.github.io/images/spring/di/AbstractBeanFactory-doGetBean4.png)

分析以上代码，我们发现有5个重点内容：

1. 优先从单例缓存中获取`Bean`实例，如果获取到了，则无需再创建对象。
2. 优先创建当前`Bean`依赖的所有`Bean`。
3. 如果是单例，创建对象之后会放到缓存中。
4. 如果是原型，每次调用都会创建新的对象。
5. 如果是自定义`scope`，则使用自定义`scope`来管理创建对象的作用域。

不管以上哪种方式，最终创建都想都调用了`createBean()`方法。

## 开始实例化

`AbstractBeanFactory`的`createBean()`最终调用了`AbstractAutowireCapableBeanFactory`的实现，源码如下：
![AbstractAutowireCapableBeanFactory-createBean](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-createBean.png)

![AbstractAutowireCapableBeanFactory-doCreateBean](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-doCreateBean.png)

在`doCreateBean()`的实现中，主要包含4个步骤：

1. 创建`Bean`实例。
2. 如果是单例，缓存单例对象，尽早持有引用，解决循环依赖问题。
3. 填充`Bean`属性，依赖注入发生在这里。
4. 初始化`Bean`，包括`afterPropertiesSet()`方法和自定义的初始化方法。

下面我们分别介绍创建`Bean`实例、填充`Bean`属性以及初始化`Bean`的具体实现。

## 创建`Bean`实例

创建`Bean`实例调用了`createBeanInstance()`方法：
![AbstractAutowireCapableBeanFactory-createBeanInstance](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-createBeanInstance.png)

在`createBeanInstance()`的实现中，主要根据不同情况来进行实例化，主要有以下四种情况：

1. 调用创建Bean的回调方法实例化。
2. 调用工厂方法实例化。
3. 使用容器的自动装配特性，调用匹配的构造方法实例化
4. 使用默认的无参构造方法实例化

前三种方式比较简单，直接调用匹配的方法实例化即可。但是对于我们最常使用的默认无参构造方法就需要使用相应的实例化策略(JDK的反射机制或者 CGLib)来进行实例化了，在方法`getInstantiationStrategy().instantiate()`中就具体实现类使用相应的策略来实例化对象。源码如下：
![AbstractAutowireCapableBeanFactory-instantiateBean](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-instantiateBean.png)

最终调用了`SimpleInstantiationStrategy`的`instantiate()`方法：
![SimpleInstantiationStrategy-instantiate](https://chentianming11.github.io/images/spring/di/SimpleInstantiationStrategy-instantiate.png)

如果`Bean`没有方法被覆盖（表示不是子类），则使用JDK的反射机制进行实例化，否则，使用CGLib进行实例化。`instantiateWithMethodInjection()`方法调用`SimpleInstantiationStrategy`的子类 `CGLibSubclassingInstantiationStrategy`使用CGLib来进行实例化，其源码如下:
![CglibSubclassingInstantiationStrategy-instantiate](https://chentianming11.github.io/images/spring/di/CglibSubclassingInstantiationStrategy-instantiate.png)

> CGLib 是一个常用的字节码生成器的类库，它提供了一系列API实现Java字节码的生成和转换功能。 我们在学习JDK的动态代理时都知道，JDK 的动态代理只能针对接口，如果一个类没有实现任何接口，要对其进行动态代理只能使用CGLib。

## 填充`Bean`属性

**在创建`Bean`实例完成之后，接下来就是填充`Bean`属性，即我们常说的依赖注入阶段了**。`populateBean()`源码如下：
![AbstractAutowireCapableBeanFactory-populateBean](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-populateBean.png)

在正式进行依赖注入之前，首先会调用`Bean`实例化后置处理器，执行实例化的后置逻辑。对于自动注入的属性，分别可以根据名称或者类型完成注入。最后调用统一的`applyPropertyValues()`方法完成属性赋值。
![AbstractAutowireCapableBeanFactory-applyPropertyValues](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-applyPropertyValues.png)

对属性的注入过程分以下两种情况:

1. 属性值类型不需要强制转换时，不需要解析属性值，直接准备进行依赖注入。
2. 属性值需要进行类型强制转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的 属性值进行依赖注入。

对属性值的解析是在`BeanDefinitionValueResolver`类中的`resolveValueIfNecessary()`方法中进行的，对属性值的依赖注入是通过 `bw.setPropertyValues()`方法实现的，在分析属性值的依赖注入之前，我们先分析一下对属性值的解析过程。

当容器在对属性进行依赖注入时，如果发现属性值需要进行类型转换，如属性值是容器中另一个`Bean`实例对象的引用，则容器首先需要根据属性值解析出所引用的对象，然后才能将该引用对象注入到目标实例对象的属性上去，对属性进行解析的由`resolveValueIfNecessary()`方法实现，其源码如下:
![BeanDefinitionValueResolver-resolveValueIfNecessary](https://chentianming11.github.io/images/spring/di/BeanDefinitionValueResolver-resolveValueIfNecessary.png)

在上面的实现中，根据不同情况，Spring将引用类型，内部类以及集合类型等属性进行解析。解析完成后就可以进行依赖注入了。依赖注入的过程就是`Bean`对象实例设置到它所依赖的`Bean`对象属性上去。依赖注入本质就是给属性赋值，这里就详细展开了。

## 初始化`Bean`

前面已经创建好`Bean`实例并且也完成了属性注入，下一步就是调用初始化方法做一些初始化操作了，源码如下：
![AbstractAutowireCapableBeanFactory-initializeBean](https://chentianming11.github.io/images/spring/di/AbstractAutowireCapableBeanFactory-initializeBean.png)

初始化方法主要包含4部分内容:

1. **调用`Aware`方法**。Spring提供可很多`Aware`接口，如果`Bean`实现了`Aware`接口，那么在初始化阶段就会调用某些`Aware`方法。这里会调用的包括`BeanNameAware`、`BeanClassLoaderAware`和`BeanFactoryAware`的方法。
2. **调用`Bean`后置处理器，在初始化之前做一些处理**。Spring提供可很多`BeanPostProcessor`接口，用于在`Bean`的各个生命周期中织入自定义逻辑。这里会执行应用的`BeanPostProcessor`的`postProcessBeforeInitialization()`方法。
3. **调用初始化方法**。包括`InitializingBean`接口的`afterPropertiesSet()`以及自定义的初始化方法。
4.  **调用`Bean`后置处理器，在初始化之后做一些处理**。这里会执行应用的`BeanPostProcessor`的`postProcessAfterInitialization()`方法。

## 总结

Spring容器启动主要分为两大阶段，第一阶段是配置资源的加载和注册，第二阶段是`Bean`的实例化、依赖注入和初始化。在第一阶段，最关注的是`BeanDefinition`，不管配置方式是xml还是注解，最终都会解析成`BeanDefinition`对象，并将其注册到容器中。在第一阶段完成之后，容器中已经有了所有`Bean`的配置信息。第二阶段则是依赖第一阶段处理好的`BeanDefinition`，从而实现了`Bean`的实例化、依赖注入和初始化操作。并且，在整个容器启动过程中，Spring提供了各种各样的回调入口，支持用户在各个执行阶段都可以注入自定义处理逻辑。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)
