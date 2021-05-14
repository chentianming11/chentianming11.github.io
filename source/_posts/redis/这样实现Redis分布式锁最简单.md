---
title: 这样实现Redis分布式锁最简单
tags: redis
categories: redis
abbrlink: 82102136
date: 2021-05-13 11:44:04
---

分布式锁是实际应用中最常见的场景之一，Redisson虽然支持了分布式锁实现，但是使用上仍然不够方便。本文介绍了一种优雅的分布式锁的静态方法封装和注解式封装实现。

<!--more-->

## 静态方法封装

为了方便在业务代码中使用分布式锁，可以通过静态方法直接调用来实现。结合函数式编程，完整接口定义如下：

```java
/**
 * 分布式锁定执行，返回执行结果
 *
 * @param key         分布式锁标识
 * @param leaseTimeMs 自动释放时间
 * @param executable    可执行单元
 * @return 执行结果
 */
public static Object lockExecute(String key, Long leaseTimeMs, Executable executable);

/**
 * 分布式锁定执行，不返回执行结果
 *
 * @param key         分布式锁标识
 * @param leaseTimeMs 自动释放时间
 * @param runnable    可执行单元
 */
public static void lockRun(String key, Long leaseTimeMs, Runnable runnable);
```

**`lockExecute()`支持返回分布式锁加锁执行结果，`lockRun()`则无返回结果**。

完整代码如下：

```java
/**
 * @author 陈添明
 */
@UtilityClass
@Slf4j
public class RLockUtils {


    /**
     * 分布式锁定执行，返回执行结果
     *
     * @param key         分布式锁标识
     * @param leaseTimeMs 自动释放时间
     * @param executable    可执行单元
     * @return 执行结果
     */
    @SneakyThrows
    public static Object lockExecute(String key, Long leaseTimeMs, Executable executable) {
        RLock lock = doLock(key, leaseTimeMs);
        try {
            return executable.execute();
        } finally {
            doUnlock(key, leaseTimeMs, lock);
        }
    }

    /**
     * 分布式锁定执行，不返回执行结果
     *
     * @param key         分布式锁标识
     * @param leaseTimeMs 自动释放时间
     * @param runnable    可执行单元
     */
    public static void lockRun(String key, Long leaseTimeMs, Runnable runnable) {
        RLock lock = doLock(key, leaseTimeMs);
        try {
            runnable.run();
        } finally {
            doUnlock(key, leaseTimeMs, lock);
        }
    }

    private static void doUnlock(String key, Long leaseTimeMs, RLock lock) {
        if (lock != null) {
            try {
                lock.unlock();
            } catch (Exception e) {
                log.error("分布式锁释放失败!不影响业务执行！key={}, leaseTimeMs={}", key, leaseTimeMs, e);
            }
        }
    }


    private static RLock doLock(String key, Long leaseTimeMs) {
        RLock lock = null;
        try {
            RedissonClient redissonClient = SpringUtil.getBean(RedissonClient.class);
            lock = redissonClient.getLock(key);
            lock.lock(leaseTimeMs, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            log.error("分布式锁加锁失败！不影响业务正常执行！key={}, leaseTimeMs={}", key, leaseTimeMs, e);
        }
        return lock;
    }

}
```

## 注解式封装

注解式编程是日常工作中更常用的一种方式，下面重点介绍注解式的实现方案。

### 定义注解`@RLock`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RLock {

    /**
     * 分布式锁标识，支持以下格式：
     * abc
     * #{param}
     * #{user.id}
     * abc_#{param}_#{user.id}
     * @return key
     */
    String key();

    /**
     * 自动释放锁时间，单位毫秒,默认5_000
     * @return leaseTime
     */
    long leaseTimeMs() default 5_000;
}
```

### 定义切面实现

```java
@Aspect
@Component
@Slf4j
@Order(0)
public class RLockAspect {

    ExpressionParser parser = new SpelExpressionParser();

    private final PropertyPlaceholderHelper defaultHelper = new PropertyPlaceholderHelper("#{", "}");

    private final ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

    @Around("@annotation(rLock)")
    public Object lock(ProceedingJoinPoint joinPoint, RLock rLock) throws Throwable {
        String key;
        try {
            key = generateKey(joinPoint, rLock);
        } catch (Exception e) {
            log.error("生成分布式锁标识失败，执行单元将会直接执行！");
            return joinPoint.proceed();
        }
        return RLockUtils.lockExecute(key, rLock.leaseTimeMs(), joinPoint::proceed);
    }


    /**
     * 生成分布式锁标识
     *
     * @param joinPoint 连接点
     * @param rLock     锁注解
     * @return 分布式锁标识
     */
    private String generateKey(ProceedingJoinPoint joinPoint, RLock rLock) {
        String key = rLock.key();
        Assert.hasText(key, "key不能配置空白字符！");
        EvaluationContext context = new StandardEvaluationContext();
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Object[] args = joinPoint.getArgs();
        String[] parameterNames = parameterNameDiscoverer.getParameterNames(method);
        if (parameterNames != null && parameterNames.length > 0) {
            for (int i = 0; i < parameterNames.length; i++) {
                context.setVariable(parameterNames[i], args[i]);
            }
        }
        return defaultHelper.replacePlaceholders(key, placeholderName -> {
            Expression expression = parser.parseExpression("#" + placeholderName);
            return String.valueOf(expression.getValue(context));
        });
    }

}

```

切面将会自动拦截所有标注了`@RLock`的方法，并织入分布式锁的代码。重点有3个`Spring`相关类需要关注一下：

- `ParameterNameDiscoverer`：用于获取方法参数名称。详细信息可参考[为何Spring MVC可获取到方法参数名，而MyBatis却不行？](https://cloud.tencent.com/developer/article/1497751)
- `PropertyPlaceholderHelper`: 用于处理占位符解析。详细信息可参考[PropertyPlaceholderHelper文档](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/PropertyPlaceholderHelper.html)。
- `ExpressionParser`： 用于解析`SpringEL`表达式。详细信息可参考[ExpressionParser文档](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/expression/ExpressionParser.html)

### 使用

在任意方法上，标注`@RLock`即可自动织入分布式锁：

```java
@RLock(key = "abc_#{id}_#{person.name}_#{person.age}")
@SneakyThrows
public void testRLock(Long id, Person person) {
    // 做业务处理
    Thread.sleep(5000L);
}
```

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~





