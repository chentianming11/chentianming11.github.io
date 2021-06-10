---
title: zookeeper客户端框架curator使用详解
tags: zookeeper
categories: zookeeper
abbrlink: 3807816107
date: 2021-05-29 14:37:09
---

**`Apache Curator`是一个比较完善的zookeeper客户端框架，通过封装的一套高级API，简化了ZooKeeper的操作，因此在实际应用中都是使用`Apache Curator`来操作zookeeper的**。

<!--more-->

源码：[https://github.com/chentianming11/zookeeper-demo](https://github.com/chentianming11/zookeeper-demo)

通过查看官方文档，可以发现Curator主要解决了三类问题：

- 封装`ZooKeeper client`与`ZooKeeper server`之间的连接处理。
- 提供了一套Fluent风格的操作API。
- 提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装。

## 引入依赖

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.12.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```

## 创建会话

### 使用静态工程方法创建客户端

```java
// 重试策略 
ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);
CuratorFramework curatorFramework = CuratorFrameworkFactory.newClient(connectionStr,5000,5000, retry);
```

newClient静态工厂方法包含四个主要参数：

| 参数名|说明 |
|------------|-----------|
| connectionString | 服务器列表，格式host1:port1,host2:port2,...|
|retryPolicy|重试策略,内建有四种重试策略,也可以自行实现RetryPolicy接口|
|sessionTimeoutMs|会话超时时间，单位毫秒，默认60000ms|
|connectionTimeoutMs|连接创建超时时间，单位毫秒，默认60000ms|

### 使用Fluent风格的Api创建客户端

```java

RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client =
                CuratorFrameworkFactory.builder()
                        .connectString(connectionInfo)
                        .sessionTimeoutMs(5000)
                        .connectionTimeoutMs(5000)
                        .retryPolicy(retryPolicy)
                        .build();

```

### 创建包含隔离命名空间的客户端

**为了实现不同的Zookeeper业务之间的隔离，需要为每个业务分配一个独立的命名空间（`NameSpace`），即指定一个Zookeeper的根路径（官方术语：为Zookeeper添加“Chroot”特性）**。例如（下面的例子）当客户端指定了独立命名空间为“/base”，那么该客户端对Zookeeper上的数据节点的操作都是基于该目录进行的。通过设置Chroot可以将客户端应用与Zookeeper服务端的一课子树相对应，在多个应用共用一个Zookeeper集群的场景下，这对于实现不同应用之间的相互隔离十分有意义。

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client =
        CuratorFrameworkFactory.builder()
                .connectString(connectionInfo)
                .sessionTimeoutMs(5000)
                .connectionTimeoutMs(5000)
                .retryPolicy(retryPolicy)
                .namespace("base")
                .build();
```

*当客户端创建成功，直接调用`start()`方法即可启动客户端*。

## 数据节点操作

### 创建数据节点

Zookeeper的节点创建模式：

- PERSISTENT：持久化节点
- PERSISTENT_SEQUENTIAL：持久化顺序节点
- EPHEMERAL：临时节点
- EPHEMERAL_SEQUENTIAL：临时顺序节点

**创建一个节点，初始内容为空**

```java
client.create().forPath("/path");
```

如果没有设置节点属性，**节点创建模式默认为持久化节点，内容默认为空**。

**创建一个节点，附带初始化内容**

```java
client.create().forPath("/path2", "test1".getBytes());
```

**创建一个节点，指定创建模式（临时节点），内容为空**

```java
client.create().withMode(CreateMode.EPHEMERAL).forPath("/path3");
```

**创建一个节点，指定创建模式（临时节点），附带初始化内容**

```java
client.create().withMode(CreateMode.EPHEMERAL).forPath("/path", "demo".getBytes());
```

**创建一个节点，指定创建模式（临时节点），附带初始化内容，并且自动递归创建父节点**

```java
client.create()
      .creatingParentContainersIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .forPath("/abc/path", "init".getBytes());
```

`creatingParentContainersIfNeeded()`接口非常有用，因为一般情况开发人员在创建一个子节点必须判断它的父节点是否存在，如果不存在直接创建会抛出`NoNodeException`，**使用`creatingParentContainersIfNeeded()`之后Curator能够自动递归创建所有所需的父节点**。

### 删除数据节点

**删除一个节点**

```java
 client.delete().forPath("/path");
```

*注意，此方法只能删除叶子节点，否则会抛出异常*。

**删除一个节点，并且递归删除其所有的子节点**

```java
client.delete()
                .deletingChildrenIfNeeded()
                .forPath("/path");
```

**删除一个节点，强制指定版本进行删除**

```java
client.delete()
      .withVersion(100)
      .forPath("/path");

```

**删除一个节点，强制保证删除**

```java
client.delete()
      .guaranteed()
      .forPath("/path");
```

*上面的多个流式接口是可以自由组合*，例如：

```java
client.delete()
      .guaranteed()
      .deletingChildrenIfNeeded()
      .withVersion(10086)
      .forPath("path");
```

### 读取数据节点数据

**读取一个节点的数据内容**

```

byte[] data = client.getData().forPath("/path2");

```

**读取一个节点的数据内容，同时获取到该节点的stat**

```java
Stat stat = new Stat();
byte[] data = client.getData()
        .storingStatIn(stat)
        .forPath("/path2");
```

### 更新数据节点数据

**更新一个节点的数据内容**

```java
client.setData().forPath("/path","data".getBytes());
```

**更新一个节点的数据内容，强制指定版本进行更新**

```java
client.setData().withVersion(10086).forPath("/path","data".getBytes());
```

### 检查节点是否存在

```java
Stat stat = client.checkExists().forPath("/path");
```

注意：该方法返回一个Stat实例，不存在，则返回null。


### 获取某个节点的所有子节点路径

```java
List<String> list = client.getChildren().forPath("/path");
```

该方法的返回值为List<String>,获得ZNode的子节点Path列表。

### 事务

CuratorFramework的实例包含`inTransaction()`接口方法，调用此方法开启一个ZooKeeper事务. 可以复合`create`, `setData`, `check`, `and/or delete` 等操作然后调用`commit()`作为一个**原子操作提交**。一个例子如下：

```java
 client.inTransaction()
                .check().forPath("/path")
                .and()
                .create().withMode(CreateMode.EPHEMERAL).forPath("/path", "data".getBytes())
                .and()
                .setData().withVersion(1000).forPath("/path", "data2".getBytes())
                .and()
                .commit();
```

上面提到的创建、删除、更新、读取等方法都是同步的，`Curator`提供异步接口，引入了`BackgroundCallback`接口用于处理异步接口调用之后服务端返回的结果信息。`BackgroundCallback`接口中一个重要的回调值为`CuratorEvent`，里面包含事件类型、响应吗和节点的详细信息。

**CuratorEventType**

| 事件类型|对应CuratorFramework实例的方法|
|------------|-----------|
|CREATE |create()|
|DELETE|delete()|
|EXISTS|checkExists()|
|GET_DATA|getData()|
|SET_DATA|setData()|
|CHILDREN|getChildren()|
|SYNC|sync(String,Object)|
|GET_ACL|getACL()|
|SET_ACL|setACL()|
|WATCHED|Watcher(Watcher)|
|CLOSING|close()|

**响应码(`getResultCode()`)**

| 响应码|意义|
|------------|-----------|
|0|OK，即调用成功|
|-4|ConnectionLoss，即客户端与服务端断开连接|
|-110|NodeExists，即节点已经存在|
|-112|SessionExpired，即会话过期|



```java
ExecutorService executor = Executors.newFixedThreadPool(2);
        client.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.EPHEMERAL)
                // 异步
// .inBackground()
                .inBackground((client1, event) -> {
                    // 事件类型
                    CuratorEventType type = event.getType();
                    // 结果编码
                    int resultCode = event.getResultCode();
                    System.out.println("事件类型: " + type + ", 结果编码: " + resultCode);
                }, executor)
                .forPath("/path");
```

注意：如果`inBackground()`方法不指定executor，那么会默认使用Curator的EventThread去进行异步处理。

## 缓存

**强烈推荐使用`ConnectionStateListener`监控连接的状态，当连接状态为LOST，curator-recipes下的所有Api将会失效或者过期**。

zookeeper原生支持通过注册`Watcher`来进行事件监听，但是开发者需要反复注册(Watcher只能单次注册单次使用)。**Cache是Curator中对事件监听的包装，可以看作是对事件监听的本地缓存视图，能够自动为开发者处理反复注册监听**。Curator提供了三种Watcher(Cache)来监听结点的变化。

- Path Cache

**Path Cache用来监控一个Znode的子节点**. 当一个子节点增加， 更新，删除时， Path Cache会改变它的状态，会包含最新的子节点，子节点的数据和状态，而状态的变更将通过`PathChildrenCacheListener`通知。

实际使用时会涉及到四个类：

- PathChildrenCache
- PathChildrenCacheEvent
- PathChildrenCacheListener
- ChildData

通过下面的构造函数创建Path Cache:

```java
public PathChildrenCache(CuratorFramework client, String path, boolean cacheData);
```

想使用cache，必须调用它的start方法，使用完后调用close方法。 可以设置StartMode来实现启动的模式。

- NORMAL：正常初始化。
- BUILD_INITIAL_CACHE：在调用start()之前会调用rebuild()。
- POST_INITIALIZED_EVENT： 当Cache初始化数据后发送一个`PathChildrenCacheEvent.Type#INITIALIZED`事件

`addListener(PathChildrenCacheListener listener)`可以增加listener监听缓存的变化。
`getCurrentData()`方法返回一个List对象，可以遍历所有的子节点。

*设置/更新、移除其实是使用client (CuratorFramework)来操作, 不通过PathChildrenCache操作*。

```java
// 创建path cache
/**
 * 注意：如果new PathChildrenCache(client, PATH, cacheData)中的参数cacheData值设置为false，
 * 则示例中的event.getData() 将返回null，cache将不会缓存节点数据。
 */
PathChildrenCache cache = new PathChildrenCache(client, PATH, true);
cache.start(PathChildrenCache.StartMode.NORMAL);

// 添加一个监听器
cache.getListenable()
        .addListener((client1, event) -> {
            // 事件类型
            PathChildrenCacheEvent.Type type = event.getType();
            System.out.println("事件类型：" + type);
            // 子节点数据
            ChildData data = event.getData();
            System.out.println("子节点数据：" + data);
        });
```

注意：如果`new PathChildrenCache(client, PATH, cacheData)`中的参数`cacheData`值设置为`false`，则示例中的`event.getData()`将返回null，cache将不会缓存节点数据。

### Node Cache

`Node Cache`与`Path Cache`类似，**`Node Cache`只是监听某一个特定的节点**。它涉及到下面的三个类：

- `NodeCache`：Node Cache实现类
- `NodeCacheListener`：节点监听器
- `ChildData`： 节点数据

`getCurrentData()`将得到节点当前的状态，通过它的状态可以得到当前的值。

```java
// 创建NodeCache
NodeCache nodeCache = new NodeCache(client, PATH);
// 必须要先start
nodeCache.start();

NodeCacheListener listener = () -> {
    System.out.println("节点变化");
    ChildData currentData = nodeCache.getCurrentData();
    if (currentData != null) {
        System.out.println("节点数据：" + currentData);
    }else {
        System.out.println("节点无数据");
    }
};
nodeCache.getListenable().addListener(listener);
```

### Tree Cache

Tree Cache可以监控整个树上的所有节点，主要涉及到下面四个类：

- `TreeCache`：Tree Cache实现类
- `TreeCacheListener`：监听器类
- `TreeCacheEvent`：触发的事件类
- `ChildData`：节点数据

```java
TreeCache treeCache = new TreeCache(client, PATH);
treeCache.start();

treeCache.getListenable().addListener((client1, event) -> {
    TreeCacheEvent.Type type = event.getType();
    ChildData data = event.getData();
    System.out.println("事件类型：" + type + ", 节点数据：" + data);
});
```

## Leader选举

在分布式计算中，`leader elections`是很重要的一个功能。这个选举过程是这样子的： 指派一个进程作为组织者，将任务分发给各节点。 在任务开始前， 哪个节点都不知道谁是leader(领导者)或者coordinator(协调者). 当选举算法开始执行后， 每个节点最终会得到一个唯一的节点作为任务leader. 除此之外，选举还经常会发生在leader意外宕机的情况下，新的leader要被选举出来。

**在zookeeper集群中，leader负责写操作，然后通过Zab协议实现follower的同步，leader或者follower都可以处理读操作**。
Curator有两种leader选举的recipe,分别是`LeaderSelector`和`LeaderLatch`。

- `LeaderSelector`：所有存活的客户端不间断的轮流做Leader。
- `LeaderLatch`：一旦选举出Leader，除非有客户端挂掉重新触发选举，否则不会交出领导权。

### LeaderLatch

`LeaderLatch`会和其它使用相同latch path的其它`LeaderLatch`交涉，然后其中一个最终会被选举为`leader`，可以通过`hasLeadership()`方法查看`LeaderLatch`实例是否leader。

类似JDK的`CountDownLatch`， `LeaderLatch`在请求成为`leadership`会`block`(阻塞)，一旦不使用`LeaderLatch`了，必须调用`close()`方法。 如果它是leader,会释放leadership， 其它的参与者将会选举一个leader。

异常处理： `LeaderLatch`实例可以增加`ConnectionStateListener`来监听网络连接问题。 当`SUSPENDED`或`LOST`时, leader不再认为自己还是leader。当`LOST`后连接重连后`RECONNECTED`, LeaderLatch会删除先前的ZNode然后重新创建一个。**LeaderLatch用户必须考虑导致leadership丢失的连接问题。 强烈推荐你使用`ConnectionStateListener`**。

```java
/**
 * 参与选举的所有节点，会创建一个顺序节点，其中最小的 节点会设置为 master 节点, 没抢到 Leader 的节点都监听 前一个节点的删除事件，
 * 在前一个节点删除后进行重新抢主，当 master 节点手动调用 close()方法或者 master 节点挂了之后，后续的子节点会抢占 master。
 其中 spark 使用的就是这种方法
 * @author 陈添明
 * @date 2018/12/14
 */
public class LeaderLatchTest {

    protected static String PATH = "/francis/leader";
    private final static String connectionInfo = "127.0.0.1:2181";
    private static final int CLIENT_QTY = 10;

    public static void main(String[] args) {
        List<CuratorFramework> clients = Lists.newArrayList();
        List<LeaderLatch> examples = Lists.newArrayList();

        try {

            /**
             * 创建10个客户端，并且创建对应的LeaderLatch
             */
            for (int i = 0; i < CLIENT_QTY; i++) {
                CuratorFramework client
                        = CuratorFrameworkFactory.newClient(connectionInfo, new ExponentialBackoffRetry(20000, 3));
                clients.add(client);
                LeaderLatch latch = new LeaderLatch(client, PATH, "Client #" + i);

                latch.addListener(new LeaderLatchListener(){
                    @Override
                    public void isLeader() {
                        System.out.println("I am Leader");
                    }

                    @Override
                    public void notLeader() {
                        System.out.println("I am not Leader");
                    }
                });

                examples.add(latch);
                client.start();
                latch.start();
            }

            Thread.sleep(10_000);
            /**
             * 获取当前的leader
             */
            LeaderLatch currentLeader = null;
            for (LeaderLatch latch : examples) {
                if (latch.hasLeadership()) {
                    currentLeader = latch;
                }
            }
            System.out.println("current leader is " + currentLeader.getId());
            System.out.println("release the leader " + currentLeader.getId());
            // 只能通过close释放当前的领导权
            currentLeader.close();

            Thread.sleep(5000);

            /**
             * 再次获取leader
             */
            for (LeaderLatch latch : examples) {
                if (latch.hasLeadership()) {
                    currentLeader = latch;
                }
            }
            System.out.println("current leader is " + currentLeader.getId());
            System.out.println("release the leader " + currentLeader.getId());
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            for (LeaderLatch latch : examples) {
                if (null != latch.getState())
                    CloseableUtils.closeQuietly(latch);
            }
            for (CuratorFramework client : clients) {
                CloseableUtils.closeQuietly(client);
            }
        }
    }
}
```

首先我们创建了10个`LeaderLatch`，启动后它们中的一个会被选举为leader。 因为选举会花费一些时间，start后并不能马上就得到leader。 通过`hasLeadership()`查看自己是否是leader， 如果是的话返回true。 可以通过`getId()`可以得到当前的leader的ID。只能通过`close()`释放当前的领导权。

### LeaderSelector

LeaderSelector使用的时候主要涉及下面几个类：

- LeaderSelector
- LeaderSelectorListener
- LeaderSelectorListenerAdapter
- CancelLeadershipException

类似`LeaderLatch`,`LeaderSelector`必须start:`leaderSelector.start()`; 一旦启动，当实例取得领导权时你的listener的`takeLeadership()`方法被调用。而`takeLeadership()`方法只有领导权被释放时才返回。 当你不再使用LeaderSelector实例时，应该调用它的`close()`方法。

**异常处理**：`LeaderSelectorListener`类继承`ConnectionStateListener`。LeaderSelector必须小心连接状态的改变。如果实例成为leader, 它应该响应`SUSPENDED`或`LOST`。 当`SUSPENDED`状态出现时，实例必须假定在重新连接成功之前, 它可能不再是leader了。 如果LOST状态出现， 实例不再是leader， `takeLeadership()`方法返回。

**重要**: 推荐处理方式是当收到`SUSPENDED` 或`LOST`时抛出`CancelLeadershipException`异常。这是会导致`LeaderSelector`实例中断并取消执行`takeLeadership`方法的异常。这非常重要， 你必须考虑扩展`LeaderSelectorListenerAdapter`。 `LeaderSelectorListenerAdapter`提供了推荐的处理逻辑。

```java
/**
 * 当实例被选为leader之后，调用takeLeadership方法进行业务逻辑处理，处理完成即释放领导权。
 * autoRequeue()方法的调用确保此实例在释放领导权后还可能获得领导权。
 * 这样保证了每个节点都可以获得领导权。
 *
 * @author 陈添明
 * @date 2019/11/10
 */
public class LeaderSelectorTest {

    public static void main(String[] args) throws InterruptedException {
        List<LeaderSelector> leaderSelectors = new ArrayList<>();
        List<CuratorFramework> clients = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            CuratorFramework client = CuratorFrameworkFactory.newClient("localhost:2181", new ExponentialBackoffRetry(1000, 3));
            client.start();
            clients.add(client);

            LeaderSelector leaderSelector = new LeaderSelector(client, "/master", new LeaderSelectorListenerAdapter() {
                @Override
                public void takeLeadership(CuratorFramework curatorFramework) throws Exception {
                    System.out.println(Thread.currentThread().getName() + " is a leader");
                    Thread.sleep(Integer.MAX_VALUE);
                }

                @Override
                public void stateChanged(CuratorFramework client, ConnectionState newState) {
                    super.stateChanged(client, newState);
                }
            });
            leaderSelectors.add(leaderSelector);
        }
        leaderSelectors.forEach(leaderSelector -> {
            leaderSelector.autoRequeue();
            leaderSelector.start();
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        System.out.println("==================");
        clients.forEach(client -> {
            client.close();
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        Thread.sleep(100 * 1000);
    }
}
```

## 分布式锁

1. 推荐使用`ConnectionStateListener`监控连接的状态，因为当连接LOST时你不再拥有锁。
2. 分布式的锁全局同步，这意味着任何一个时间点不会有两个客户端都拥有相同的锁。

### 可重入共享锁—Shared Reentrant Lock

`Shared`意味着锁是全局可见的，客户端都可以请求锁。`Reentrant`和JDK的`ReentrantLock`类似，即可重入。意味着**同一个客户端在拥有锁的同时，可以多次获取，不会被阻塞**。 它是由类`InterProcessMutex`来实现。 它的构造函数为：`public InterProcessMutex(CuratorFramework client, String path)`。

- 通过`acquire()`获得锁，并提供超时机制。
- 通过`release()`方法释放锁。`InterProcessMutex`实例可以重用。

zookeeper还提供了可协商的撤销机制，通过为`mutex`设置*撤销监听器*来支持撤销`mutex`, 通过调用`makeRevocable(RevocationListener<T> listener)`来实现。

如果你请求撤销当前的锁，可以调用`Revoker.attemptRevoke(CuratorFramework client, String path)`方法,此时`RevocationListener`将会回调。

**代码示例**：

首先让我们创建一个模拟的共享资源， 这个资源期望只能单客户端的访问，否则会有并发问题。

```java
/**
 * 共享资源
 */
public class FakeLimitedResource {
    private final AtomicBoolean inUse = new AtomicBoolean(false);
    private final Random random = new Random(System.currentTimeMillis());

    /**
     * use方法最多只能有一个客户端调用
     * 否则，抛出异常
     * @throws InterruptedException
     */
    public void use() throws InterruptedException {
        // 真实环境中我们会在这里访问/维护一个共享的资源
        //这个例子在使用锁的情况下不会非法并发异常IllegalStateException
        //但是在无锁的情况由于sleep了一段时间，很容易抛出异常
        if (!inUse.compareAndSet(false, true)) {
            throw new IllegalStateException("同一时间，只能被一个客户端访问！！！");
        }
        try {
            Thread.sleep(random.nextInt(2_000));
        } finally {
            inUse.set(false);
        }
    }
}
```

然后创建一个`InterProcessMutexDemo`类， 它负责请求锁， 使用资源，释放锁这样一个完整的访问过程。

```java
/**
 *
 * 然后创建一个InterProcessMutexDemo类， 它负责请求锁， 使用资源，释放锁这样一个完整的访问过程。
 * @author 陈添明
 * @date 2018/12/15
 */

public class InterProcessMutexDemo {


    /**
     * 代码也很简单，生成10个client， 每个client重复执行10次 请求锁–访问资源–释放锁的过程。
     * 每个client都在独立的线程中。 结果可以看到，锁是随机的被每个实例排他性的使用。
     */

    private InterProcessMutex lock;
    private final FakeLimitedResource resource;
    private final String clientName;

    public InterProcessMutexDemo(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName) {
        this.resource = resource;
        this.clientName = clientName;
        this.lock = new InterProcessMutex(client, lockPath);
        // 将锁设为可撤销的. 当别的进程或线程想让你释放锁时Listener会被调用。
        lock.makeRevocable((forLock -> {

        }));
    }


    /**
     * 同一线程再次 acquire，首先判断当前的 映射表内(threadData)是否有该线程的锁信息，如果有 则原子+1，然后返回
     * 可重入互斥锁
     * 加锁执行
     * @param time
     * @param unit
     * @throws Exception
     */
    public void doWork(long time, TimeUnit unit) throws Exception {
        if (!lock.acquire(time, unit)) {
            throw new IllegalStateException(clientName + "获取互斥锁锁失败");
        }
        try {
            System.out.println(clientName + " 获取到互斥锁");
            resource.use(); //access resource exclusively
        } finally {
            System.out.println(clientName + " 释放互斥锁");
            lock.release(); // always release the lock in a finally block
        }
    }

    private static final int QTY = 5;
    private static final int REPETITIONS = QTY * 10;
    private static final String PATH = "/examples/locks";

    public static void main(String[] args) throws Exception {
        final FakeLimitedResource resource = new FakeLimitedResource();
        ExecutorService service = Executors.newFixedThreadPool(QTY);
        final TestingServer server = new TestingServer();
        try {
            for (int i = 0; i < QTY; ++i) {
                final int index = i;
                Callable<Void> task = () -> {
                    // 创建客户端
                    CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
                    try {
                        // 启动
                        client.start();
                        final InterProcessMutexDemo example = new InterProcessMutexDemo(client, PATH, resource, "Client " + index);
                        // 执行
                        for (int j = 0; j < REPETITIONS; ++j) {
                            example.doWork(10, TimeUnit.SECONDS);
                        }
                    } catch (Throwable e) {
                        e.printStackTrace();
                    } finally {
                        CloseableUtils.closeQuietly(client);
                    }
                    return null;
                };
                service.submit(task);
            }
            service.shutdown();
            service.awaitTermination(10, TimeUnit.MINUTES);
        } finally {
            CloseableUtils.closeQuietly(server);
        }
    }

}
```

### 不可重入共享锁—Shared Lock

这个锁和上面的`InterProcessMutex`相比，就是少了`Reentrant`的功能，也就意味着它不能在同一客户端中重入。这个类是`InterProcessSemaphoreMutex`,使用方法和`InterProcessMutex`类似

> 源码见`InterProcessSemaphoreMutexDemo`类。

运行后发现，有且只有一个client成功获取第一个锁(第一个`acquire()`方法返回true)，然后它自己阻塞在第二个`acquire()`方法，获取第二个锁超时；其他所有的客户端都阻塞在第一个`acquire()`方法超时并且抛出异常。
这样也就验证了`InterProcessSemaphoreMutex`实现的锁是不可重入的。

### 可重入读写锁—Shared Reentrant Read Write Lock

类似JDK的`ReentrantReadWriteLock`，读写锁一个负责读操作，另外一个负责写操作。读操作在写锁没被使用时可同时由多个进程使用，而写锁在使用时不允许读(阻塞)。
此锁是可重入的。一个拥有写锁的线程可重入读锁，但是读锁却不能进入写锁。这也意味着写锁可以降级成读锁， 比如`请求写锁 --->请求读锁--->释放读锁 ---->释放写锁`。从读锁升级成写锁是不行的。
可重入读写锁主要由两个类实现：`InterProcessReadWriteLock`、`InterProcessMutex`。使用时首先创建一个`InterProcessReadWriteLock`实例，然后再根据你的需求得到读锁或者写锁，读写锁的类型是`InterProcessMutex`。

### 信号量—Shared Semaphore

一个计数的信号量类似JDK的`Semaphore`。 JDK中`Semaphore`维护的一组许可(permits)，而`Curator`中称之为租约(Lease)。 有两种方式可以决定`semaphore`的最大租约数。第一种方式是用户给定`path`并且指定最大`LeaseSize`。第二种方式用户给定path并且使用`SharedCountReader`类。如果不使用`SharedCountReader`, 必须保证所有实例在多进程中使用相同的(最大)租约数量,否则有可能出现A进程中的实例持有最大租约数量为10，但是在B进程中持有的最大租约数量为20，此时租约的意义就失效了。

**调用`acquire()`会返回一个租约对象。 客户端必须在`finally`中`close`这些租约对象，否则这些租约会丢失掉**。 但是，如果客户端`sessio`n由于某种原因比如`crash`丢掉， 那么这些客户端持有的租约会自动`close`， 这样其它客户端可以继续使用这些租约。 租约还可以通过下面的方式返还：

```java
public void returnAll(Collection<Lease> leases)
public void returnLease(Lease lease)
```

**注意你可以一次性请求多个租约，如果`Semaphore`当前的租约不够，则请求线程会被阻塞**。 同时还提供了超时的重载方法。

```java
public Lease acquire()
public Collection<Lease> acquire(int qty)
public Lease acquire(long time, TimeUnit unit)
public Collection<Lease> acquire(int qty, long time, TimeUnit unit)
```

`Shared Semaphore`使用的主要类包括下面几个： `InterProcessSemaphoreV2`、`Lease`、`SharedCountReader`。
使用示例代码如下：

```java
public class InterProcessSemaphoreDemo {


    private static final int MAX_LEASE = 10;
    private static final String PATH = "/examples/locks";

    /**
     * 首先我们先获得了5个租约， 最后我们把它还给了semaphore。 接着请求了一个租约，
     * 因为semaphore还有5个租约，所以请求可以满足，返回一个租约，还剩4个租约。
     * 然后再请求5个租约，因为租约不够，阻塞到超时，还是没能满足，返回结果为null(租约不足会阻塞到超时，然后返回null，不会主动抛出异常；如果不设置超时时间，会一致阻塞)。
     *
     上面说讲的锁都是公平锁(fair)。 总ZooKeeper的角度看， 每个客户端都按照请求的顺序获得锁，不存在非公平的抢占的情况。
     * @param args
     * @throws Exception
     */

    public static void main(String[] args) throws Exception {
        FakeLimitedResource resource = new FakeLimitedResource();
        try (TestingServer server = new TestingServer()) {

            // 创建客户端并启动
            CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
            client.start();

            // 信号量
            InterProcessSemaphoreV2 semaphore = new InterProcessSemaphoreV2(client, PATH, MAX_LEASE);

            // 获取5个租约
            Collection<Lease> leases = semaphore.acquire(5);
            System.out.println("get " + leases.size() + " leases");

            // 获取1个租约
            Lease lease = semaphore.acquire();
            System.out.println("get another lease");

            // 处理资源
            resource.use();

            // 在获取5个租约，由于最多只有10个，前面已经使用了6个，租约不足。
            // 如果在获得所有租赁之前时间到期，则获得的租赁子集将自动关闭
            Collection<Lease> leases2 = semaphore.acquire(5, 10, TimeUnit.SECONDS);
            System.out.println("Should timeout and acquire return " + leases2);

            // 返还一个租约
            System.out.println("return one lease");
            semaphore.returnLease(lease);

            // 返还多个租约
            System.out.println("return another 5 leases");
            semaphore.returnAll(leases);
        }
    }
}
```

### 多共享锁对象 —Multi Shared Lock

**`Multi Shared Lock`是一个锁的容器。 当调用`acquire()`， 所有的锁都会被`acquire()`，如果请求失败，所有的锁都会被`release`。 同样调用`release`时所有的锁都被`release`(失败被忽略)**。 基本上，它就是组锁的代表，在它上面的请求释放操作都会传递给它包含的所有的锁。
主要涉及两个类：`InterProcessMultiLock`、`InterProcessLock`。 它的构造函数需要包含的锁的集合，或者一组ZooKeeper的path。

```java
public InterProcessMultiLock(List<InterProcessLock> locks)
public InterProcessMultiLock(CuratorFramework client, List<String> paths)
```

## 共享 shared

### 共享计数—SharedCount

这个类使用`int`类型来计数。 主要涉及三个类。`SharedCount`、`SharedCountReader`、`SharedCountListener`。
`SharedCount`代表计数器， 可以为它增加一个`SharedCountListener`，当计数器改变时此`Listener`可以监听到改变的事件，而`SharedCountReader`可以读取到最新的值， 包括字面值和带版本信息的值`VersionedValue`。

## 分布式原子类-atomic

### DistributedAtomicLong

顾名思义，计数器是用来计数的, 利用ZooKeeper可以实现一个集群共享的计数器。 只要使用相同的`path`就可以得到最新的计数器值， 这是由ZooKeeper的一致性保证的。
除了计数的范围比`SharedCount`大了之外， 它首先尝试使用乐观锁的方式设置计数器， 如果不成功(比如期间计数器已经被其它client更新了)， 它使用`InterProcessMutex`方式来更新计数值。

此计数器有一系列的操作：
- `get()`: 获取当前值
- `increment()`： 加一
- `decrement()`: 减一
- `add()`： 增加特定的值
- `subtract()`: 减去特定的值
- `trySet()`: 尝试设置计数值
- `forceSet()`: 强制设置计数值

## 分布式屏障—Barrier

分布式`Barrier`是这样一个类： 它会阻塞所有节点上的等待进程，直到某一个被满足， 然后所有的节点继续进行。
比如赛马比赛中， 等赛马陆续来到起跑线前。 一声令下，所有的赛马都飞奔而出。

### DistributedBarrier

`DistributedBarrier`类实现了栅栏的功能。 它的构造函数为`public DistributedBarrier(CuratorFramework client, String barrierPath)`。

1. 首先你需要设置栅栏，它将阻塞在它上面等待的线程:`setBarrier()`。
2. 然后需要阻塞的线程调用方法等待放行条件:` waitOnBarrier()`。
3. 当条件满足时，移除栅栏，所有等待的线程将继续执行：`removeBarrier()`。

异常处理`DistributedBarrier`会监控连接状态，当连接断掉时`waitOnBarrier()`方法会抛出异常。

```java
public class DistributedBarrierDemo {

    private static final int QTY = 5;
    private static final String PATH = "/examples/barrier";


    /**
     * 这个例子创建了controlBarrier来设置栅栏和移除栅栏。 我们创建了5个线程，在此Barrier上等待。
     * 最后移除栅栏后所有的线程才继续执行。
     * @param args
     * @throws Exception
     */

    public static void main(String[] args) throws Exception {
        try (TestingServer server = new TestingServer()) {
            CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
            client.start();
            ExecutorService service = Executors.newFixedThreadPool(QTY);

            // 创建barrier
            DistributedBarrier controlBarrier = new DistributedBarrier(client, PATH);
            // 设置barrier
            controlBarrier.setBarrier();

            for (int i = 0; i < QTY; ++i) {
                final DistributedBarrier barrier = new DistributedBarrier(client, PATH);
                final int index = i;
                Callable<Void> task = () -> {
                    Thread.sleep((long) (3 * Math.random()));
                    System.out.println("Client #" + index + " waits on Barrier");
                    barrier.waitOnBarrier();
                    System.out.println("Client #" + index + " begins");
                    return null;
                };
                service.submit(task);
            }
            Thread.sleep(10000);
            System.out.println("all Barrier instances should wait the condition");
            // 放行
            controlBarrier.removeBarrier();
            service.shutdown();
            service.awaitTermination(10, TimeUnit.MINUTES);

            Thread.sleep(20000);
        }
    }
}
```

### 双栅栏—DistributedDoubleBarrier

双栅栏允许客户端在计算的开始和结束时同步。当足够的进程加入到双栅栏时，进程开始计算， 当计算完成时，离开栅栏。 双栅栏类是`DistributedDoubleBarrier`。 构造函数为:

```java
public DistributedDoubleBarrier(CuratorFramework client,
                                String barrierPath,
                                int memberQty)
```

`memberQty`是成员数量，当`enter()`方法被调用时，成员被阻塞，直到所有的成员都调用了`enter()`。 当`leave()`方法被调用时，它也阻塞调用线程，直到所有的成员都调用了`leave()`。 就像百米赛跑比赛， 发令枪响， 所有的运动员开始跑，等所有的运动员跑过终点线，比赛才结束。

```java
public class DistributedDoubleBarrierDemo {

    private static final int QTY = 5;
    private static final String PATH = "/examples/barrier";

    public static void main(String[] args) throws Exception {
        try (TestingServer server = new TestingServer()) {
            CuratorFramework client = CuratorFrameworkFactory.newClient(server.getConnectString(), new ExponentialBackoffRetry(1000, 3));
            client.start();
            ExecutorService service = Executors.newFixedThreadPool(QTY);
            for (int i = 0; i < QTY; ++i) {

                // 创建双栅栏
                final DistributedDoubleBarrier barrier = new DistributedDoubleBarrier(client, PATH, QTY);
                final int index = i;
                Callable<Void> task = () -> {

                    Thread.sleep((long) (3 * Math.random()));
                    System.out.println("Client #" + index + " enters");
                    barrier.enter();
                    System.out.println("Client #" + index + " begins");
                    Thread.sleep((long) (3000 * Math.random()));
                    barrier.leave();
                    System.out.println("Client #" + index + " left");
                    return null;
                };
                service.submit(task);
            }

            service.shutdown();
            service.awaitTermination(10, TimeUnit.MINUTES);
            Thread.sleep(Integer.MAX_VALUE);
        }
    }

}
```

> 欢迎关注我的开源项目：[一款适用于SpringBoot的轻量级HTTP调用框架](https://github.com/LianjiaTech/retrofit-spring-boot-starter)

