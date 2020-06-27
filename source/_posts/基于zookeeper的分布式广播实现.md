---
title: 基于zookeeper的分布式广播组件实现详解
tags: zookeeper
categories: 中间件
abbrlink: 2818246864
date: 2020-06-27 19:11:33
---

在实际的业务开发中，很多时候需要在同一应用不同节点之间进行事件广播。众所周知，`zookeeper`又是一个非常好并且应用广泛的分布式协调组件，因此`zookeeper`非常适合用来实现上述功能。本文主要讲述基于`curator`（一个`zookeeper`客户端）的分布式广播组件实现。

<!--more-->

## 实现方案详解

为了大家能更方便的使用，组件API参考了`guava`的`eventBus`。废话不多说，直接上实现代码。

> `curator`与spring项目整合不属于本文内容，需要了解的同学可自行了解。

### 广播订阅注解`@BroadcastSubscribe`

标注在订阅方法上，被`@BroadcastSubscribe`标记的方法收到广播事件时就会执行。

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface BroadcastSubscribe {
}

```

### 广播组件实现`Broadcast`

实现了广播事件的注册，发布等逻辑。因为实际项目都是spring项目，因此直接与spring项目整合。

```java
@Component
@Slf4j
public class Broadcast implements ApplicationListener<ApplicationReadyEvent> {

    @Autowired
    private CuratorFramework curatorFramework;

   private static final String BASE_PATH = "/broadcast/";

   private Set<SubscribeMapping> copyOnWriteArraySet = new CopyOnWriteArraySet<>();

    @Data
    @Accessors(chain = true)
    public static class SubscribeMapping {

        private String path;

        private Method method;

        private Object instance;
    }

    /**
     * 注册订阅制对象，将订阅者对象中被`@BroadcastSubscribe`标注的方法进行注册
     * @param subscribe 订阅者对象
     */
    public void register(@NotNull Object subscribe) {
        Class<?> clz = subscribe.getClass();
        Method[] methods = clz.getMethods();
        for (Method method : methods) {
            if (!method.isAnnotationPresent(BroadcastSubscribe.class)){
                continue;
            }
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length != 1) {
                String error = String.format("the method[%s] parameter length must equals 1", method.getName());
                throw new IllegalArgumentException(error);
            }
            Class<?> parameterType = parameterTypes[0];
            String path = BASE_PATH + parameterType.getName().replaceAll("\\.", "/");
            copyOnWriteArraySet.add(new SubscribeMapping().setPath(path).setMethod(method).setInstance(subscribe));
            log.info("broadcast subscribe mapped : {} - {}", path, method.getName());
        }
    }


    /**
     * 发布广播事件
     * @param event 广播事件
     */
    public void post(@NotNull Object event)  {
        try {
            String path = BASE_PATH + event.getClass().getName().replaceAll("\\.", "/");
            String jsonString = JSON.toJSONString(event);
            byte[] data = jsonString.getBytes();
            log.info("send broadcast, name = {}, data= {}", path, jsonString);
            if (curatorFramework.checkExists().forPath(path) == null) {
                curatorFramework.create().creatingParentContainersIfNeeded().forPath(path, data);
            } else {
                curatorFramework.setData().forPath(path, data);
            }
        } catch (Exception e) {
            log.error("send broadcast fail, event = " + event, e);
        }
    }

    /**
     * 注册zookeeper监听
     */
    private void initListener() {
        copyOnWriteArraySet.forEach(subscribeMapping -> {
            String path = subscribeMapping.getPath();
            Method method = subscribeMapping.getMethod();
            Object instance = subscribeMapping.getInstance();
            NodeCache nodeCache = new NodeCache(curatorFramework, path);
            ListenerContainer<NodeCacheListener> listenable = nodeCache.getListenable();
            listenable.addListener(() -> {
                try {
                    ChildData childData = nodeCache.getCurrentData();
                    String dataStr = new String(childData.getData());
                    Class<?> parameterType = method.getParameterTypes()[0];
                    Object param = JSON.parseObject(dataStr, parameterType);
                    log.info("receive broadcast, mame = {}, data ={} ", path, dataStr);
                    method.invoke(instance, new Object[]{param});
                } catch (Exception e) {
                    log.error("", e);
                }
            });
            try {
                nodeCache.start(true);
            } catch (Exception e) {
                log.error("nodeCache start fail !", e);
            }
        });
    }

    /**
     * Handle an application event.
     *
     * @param event the event to respond to
     */
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        initListener();
    }
}
```

## 项目使用

### 定义广播事件

```java
@Data
@Accessors(chain = true)
public class BroadcastEvent {
    private Long timeStamp;
    private Long userId;
    private String userName;
}
```

### 发布广播事件

```java
@Service
public class DemoService {

    @Autowired
    private Broadcast broadcast;


    public void test() {
        // 业务逻辑
        BroadcastEvent broadcastEvent =  new BroadcastEvent()
            .setTimeStamp(System.currentTimeMillis())
            .setUserId(100L)
            .setUserName("haha");
        // 发布广播事件
        broadcast.post(broadcastEvent);
    }
}

```

### 接收广播事件

```java
@Component
public class DemoBroadcastSubscribe {

    @Autowired
    Broadcast broadcast;

    @PostConstruct
    public void register() {
        broadcast.register(this);
    }

    @BroadcastSubscribe
    public void searchTerminologyChange(BroadcastEvent broadcastEvent) {
        // 广播事件处理逻辑
    }
}
```

至此，基于`zookeeper`的分布式广播组件实现就完成了！
