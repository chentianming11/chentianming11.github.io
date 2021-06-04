---
title: Spring钩子方法和钩子接口使用
tags: spring
categories: spring
abbrlink: 1422921172
date: 2021-06-02 08:01:07
---

`Spring`提供了非常多的钩子方法和钩子接口，善用它们可以极大地方便我们开发，本篇文章将会详细介绍各种钩子方法和钩子接口的使用方式。

<!--more-->

## Aware接口

**Spring中提供了各种Aware接口，方便从上下文中获取当前的运行环境**，比较常见的几个子接口有： `BeanFactoryAware`,`BeanNameAware`,`ApplicationContextAware`,`EnvironmentAware`，`BeanClassLoaderAware`等，这些`Aware`的作用都可以从命名得知，并且其使用也是十分简单。


| 接口名 | 描述 |
| --- | --- |
|`ApplicationContextAware`| 实现了这个接口的类都可以获取到一个 ApplicationContext 对象. 可以获取容器中的所有`Bean`。|
|`ApplicationEventPublisherAware`|在`bean`中可以得到应用上下文的事件发布器, 从而可以在`Bean`中发布应用上下文的事件。|
|`BeanClassLoaderAware`|获取`bean`的类加载器|
|`BeanFactoryAware`|获取`bean`的工厂|
|`BeanNameAware`|获取`bean`在容器中的名字|
|`BootstrapContextAware`|获取`BootstrapContext`|
|`LoadTimeWeaverAware`|加载Spring `Bean`时织入第三方模块, 如`AspectJ`|
|`MessageSourceAware`|主要用于获取国际化相关接口|
|`NotificationPublisherAware`|用于获取通知发布者|
|`ResourceLoaderAware`|初始化时注入`ResourceLoader`|
|`ServletConfigAware`|web开发过程中获取`ServletConfig`|
|`ServletContextAware`|web开发过程中获取`ServletContext`信息|
|`EnvironmentAware`|获取当前的应用的环境，从而用来获取配置信息、当前`profile`等信息|
|`EmbeddedValueResolverAware`|获取`StringValueResolver`，用来解析 `${app.test}` 的字符串值。|

## InitializingBean接口和DisposableBean接口

1. `InitializingBean`接口只有一个方法`#afterPropertiesSet`：当一个Bean实现`InitializingBean`，`#afterPropertiesSet`方法里面可以添加自定义的初始化方法或者做一些资源初始化操作。当`BeanFactory`设置完所有的Bean属性之后才会调用`#afterPropertiesSet`方法

2. `DisposableBean`接口只有一个方法`#destroy`：当一个单例Bean实现`DisposableBean`，`#destroy`可以添加自定义的一些销毁方法或者资源释放操作。

使用示例：

```java
@Component
public class ConcreteBean implements InitializingBean,DisposableBean {
    @Override
    public void destroy() throws Exception {
        System.out.println("释放资源");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("初始化资源");
    }
}
```

## ImportBeanDefinitionRegistrar接口

1. 当处理Java编程式配置类(使用了@Configuration的类)的时候，`ImportBeanDefinitionRegistrar`接口的实现类可以注册额外的`bean definitions`。
   
2. `ImportBeanDefinitionRegistrar`接口的实现类必须提供给`@Import`注解或者是`ImportSelector`接口返回值。
   
3. `ImportBeanDefinitionRegistrar`接口的实现类可能还会实现下面`org.springframework.beans.factory.Aware`接口中的一个或者多个，它们各自的方法优先于`ImportBeanDefinitionRegistrar#registerBeanDefinitions`被调用。
    - `org.springframework.context.EnvironmentAware`(读取或者修改Environment的变量)
    - `org.springframework.beans.factory.BeanFactoryAware` (获取Bean自身的Bean工厂)
    - `org.springframework.beans.factory.BeanClassLoaderAware`(获取Bean自身的类加载器)
    - `org.springframework.context.ResourceLoaderAware`(获取Bean自身的资源加载器)

## BeanPostProcessor接口和BeanFactoryPostProcessor接口

**一般我们叫这两个接口为Spring的Bean后置处理器接口,作用是为Bean的各个处理操作前后提供可扩展的空间**。

### BeanFactoryPostProcessor

```java
public interface BeanFactoryPostProcessor {
 void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;  
}
```

**在标准初始化之后修改应用程序上下文的内部bean工厂。 此时所有bean定义都已经加载完成，但尚未实例化任何bean。 这允许覆盖或添加属性，甚至是初始化bean**。


### BeanPostProcessor

BeanPostProcessor接口是最顶层的接口，接口定义：

```java
public interface BeanPostProcessor {
    // 初始化之前执行
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    // 初始化之后执行
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

1. `postProcessBeforeInitialization`是指bean在初始化之前需要调用的方法
2. `postProcessAfterInitialization`是指bean在初始化之后需要调用的方法
3. `postProcessBeforeInitialization`和`postProcessAfterInitialization`方法被调用的时候。这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean。

### InstantiationAwareBeanPostProcessor

`InstantiationAwareBeanPostProcessor`接口继承自`BeanPostProcessor`接口。多出了3个方法：

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
    // 在目标 bean 被实例化之前应用这个 BeanPostProcessor。 返回的 bean 对象可能是一个代理来代替目标 bean，有效地抑制目标 bean 的默认实例化。
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
       return null;
    }
    
    // 在 bean 被实例化之后，通过构造函数或工厂方法，但在 Spring 属性填充（从显式属性或自动装配）发生之前执行操作。
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
       return true;
    }

   // 在工厂将给定的属性值应用于给定的 bean 之前对给定的属性值进行后处理，无需任何属性描述符。
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {

       return null;
    }
}

```

1. `postProcessBeforeInstantiation`方法是最先执行的方法，它在目标对象实例化之前调用，该方法的返回值类型是`Object`，我们可以返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(比如代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只有`postProcessAfterInitialization`方法会调用，其它方法不再调用。
2. `postProcessAfterInstantiation`方法在目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是`null`。如果该方法返回`false`，会忽略属性值的设置；如果返回`true`，会按照正常流程设置属性值
3. `postProcessProperties`方法对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)。如果`postProcessAfterInstantiation`方法返回`false`，该方法不会被调用。可以在该方法内对属性值进行修改。

### SmartInstantiationAwareBeanPostProcessor

`SmartInstantiationAwareBeanPostProcessor`接口继承`InstantiationAwareBeanPostProcessor`接口。多出了3个方法：

```java
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {

   // 预测最终从此处理器的 postProcessBeforeInstantiation 回调返回的 bean 的类型。
   @Nullable
   default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
      return null;
   }

   // 确定用于给定 bean 的候选构造函数。
   @Nullable
   default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
           throws BeansException {

      return null;
   }

   // 获取对指定 bean 的早期访问的引用，通常用于解析循环引用。
   default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
      return bean;
   }

}
```

1. `predictBeanType`方法用于预测Bean的类型，返回第一个预测成功的`Class`类型，如果不能预测返回null。主要在于`BeanDefinition`无法确定`Bean`类型的时候调用该方法来确定类型。
2. `determineCandidateConstructors`方法用于选择合适的构造器，比如类有多个构造器，可以实现这个方法选择合适的构造器并用于实例化对象。
3. `getEarlyBeanReference`主要用于解决循环引用问题。只有单例对象才会调用此方法。

### DestructionAwareBeanPostProcessor

`DestructionAwareBeanPostProcessor`接口继承`BeanPostProcessor`接口。多出了2个方法：

```java
public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {

   // 在销毁之前将此 BeanPostProcessor 应用于给定的 bean 实例，例如 调用自定义销毁回调。
   void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;

   // 确定给定的 bean 实例是否需要此后处理器销毁。
   default boolean requiresDestruction(Object bean) {
      return true;
   }

}

```

### MergedBeanDefinitionPostProcessor

`DestructionAwareBeanPostProcessor`接口继承`BeanPostProcessor`接口。多出了2个方法：

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

	// 对指定 bean 的给定合并 bean 定义进行后处理。
	void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

	// 指定名称的 bean 定义已重置的通知，并且此后处理器应清除受影响 bean 的所有元数据。
	default void resetBeanDefinition(String beanName) {
	}

}
```

### BeanDefinitionRegistryPostProcessor

`BeanDefinitionRegistryPostProcessor` 接口可以看作是`BeanFactoryPostProcessor`和`ImportBeanDefinitionRegistrar`的功能集合，**既可以获取和修改`BeanDefinition`的元数据，也可以实现`BeanDefinition`的注册、移除等操作**。

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    // 在标准初始化之后修改应用程序上下文的内部 bean 定义注册表。 所有常规 bean 定义都将被加载，但尚未实例化任何 bean。 这允许在下一个后处理阶段开始之前添加更多的 bean 定义。
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```

## FactoryBean接口

首先第一眼要注意，是`FactoryBean`接口而不是`BeanFactory`接口。一般情况下，Spring通过反射机制利用`bean`的`class`属性指定实现类来实例化bean ，实例化bean过程比较复杂。**`FactoryBean`接口就是为了简化此过程，把`bean`的实例化定制逻辑下发给使用者**。

在该接口中还定义了以下3个方法。

1. `T getObject()`：返回由`FactoryBean`创建的bean实例，如果`isSingleton()``返回true`，则该实例会放到Spring容器中单实例缓存池中。
2. `boolean isSingleton()`：返回由`FactoryBean`创建的bean实例的作用域是`singleton`还是`prototype`。
3. `Class<T> getObjectType()`：返回`FactoryBean`创建的`bean`类型。

**注意一点：通过Spring容器的`getBean()`方法返回的不是`FactoryBean`本身，而是`FactoryBean#getObject()`方法所返回的对象，相当于`FactoryBean#getObject()`代理了`getBean()`方法。如果希望获取`FactoryBean`的实例，则需要在使用`getBean`(beanName) 方法时在`beanName`前显示的加上 "&" 前缀。**

## ApplicationListener

`ApplicationListener`是一个接口，里面只有一个`onApplicationEvent(E event)`方法，这个泛型`E`必须是`ApplicationEvent`的子类，而`ApplicationEvent`是Spring定义的事件，继承于`EventObject`，构造要求必须传入一个`Object`类型的`source`，这个`source`可以作为一个存储对象。将会在`ApplicationListener`的`onApplicationEvent`里面得到回调。如果在上下文中部署一个实现了`ApplicationListener`接口的bean，那么每当在一个`ApplicationEvent`发布到 `ApplicationContext`时，这个`bean`得到通知。其实这就是标准的`Oberver`设计模式。另外，`ApplicationEvent`的发布由`ApplicationContext`通过`#publishEvent`方法完成。其实这个实现从原理和代码上看都有点像`Guava`的`eventbus`。

贴一个例子：
EmailEvent:
```java
@Data
public class EmailEvent extends ApplicationEvent {
    private String author;
    private String content;
    private String date;
}
```
EmailApplicationListener：
```java
@Component
public class EmailApplicationListener implements ApplicationListener<EmailEvent> {
    @Override
    public void onApplicationEvent(EmailEvent event) {
        System.out.println("EmailApplicationListener callback!!");
        System.out.println("EmailEvent --> source: " + event.getSource());
        System.out.println("EmailEvent --> author: " + event.getAuthor());
        System.out.println("EmailEvent --> content: " + event.getContent());
        System.out.println("EmailEvent --> date: " + event.getDate());
    }
}
```
测试类：
```java
@SpringBootTest(classes = Application.class)
@RunWith(SpringJUnit4ClassRunner.class)
public class EmailApplicationListenerTest {
    @Autowired
    private ApplicationContext applicationContext;
    @Test
    public void onApplicationEvent() throws Exception {
        applicationContext.publishEvent(new EmailEvent("this is source",
                "throwable","here is emailEvent","2017-5-16"));
    }
}
```











