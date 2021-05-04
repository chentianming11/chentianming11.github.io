---
title: 深入理解zookeeper-zookeeper基本介绍及使用
tags: zookeeper
categories: zookeeper
abbrlink: 4128331822
date: 2021-05-04 10:51:29
---

**zookeeper是一种为分布式应用所设计的高可用、高性能且一致的开源协调服务，主要可用来实现分布式锁、配置维护、分布式消息队列、分布式通知/协调等**。

zookeeper设计了一种全新的数据结构(`Znode`)，并且在`Znode`上定义了一些原语（关于该数据结构的一些操作）。因为zookeeper是工作在一个分布式的环境下，所以还需要一个通知机制(**watch机制**)来将消息通过网络发送给分布式应用程序。总结一下，**zookeeper所提供的服务主要是通过：Znode数据结构+原语+watch机制，三个部分来实现的**，那么接下来就从这三个方面来简单介绍一下zookeeper。
<!--more-->

源码：[https://github.com/chentianming11/zookeeper-demo](https://github.com/chentianming11/zookeeper-demo)

## zookeeper数据模型

### Znode

zookeeper拥有一个层次的命名空间，这个和标准的文件系统非常相似，如下图所示。
![](../../../public/images/zookeeper/znode.png)
![](../../../public/images/zookeeper/znode-file.png)

从图中我们可以看出，zookeeper的数据模型在结构上和标准文件系统的非常相似，都是采用**树形层次结构**，zookeeper树中的每个节点被称为—`Znode`。和文件系统的目录树一样，zookeeper树中的每个节点可以拥有子节点。当然，也有不同之处：


- 引用方式
  
  Zonde通过路径引用，如同Unix中的文件路径。**路径必须是绝对且唯一的**，因此他们必须由斜杠字符来开头，并且每一个路径只有一个表示。

- Znode结构
  
  zookeeper命名空间中的Znode，兼具文件和目录两种特点。**既像文件一样维护着数据、元信息、ACL、时间戳等数据结构，又像目录一样可以作为路径标识的一部分**。图中的每个节点称为一个Znode。 每个Znode由3部分组成:
    - stat：状态信息, 描述该Znode的版本, 权限等信息；
    - data：与该Znode关联的数据；
    - children：该Znode下的子节点
    
    zookeeper虽然可以关联一些数据，但并没有被设计为常规的数据库或者大数据存储，相反的是，它用来管理调度数据，比如分布式应用中的配置文件信息、状态信息、汇集位置等等。这些数据的共同特性就是它们都是很小的数据，通常以KB为大小单位。zookeeper的服务器和客户端都被设计为严格检查并限制每个Znode的数据大小至多1M，但常规使用中应该远小于此值。

- 数据访问

    zookeeper中的每个节点存储的数据要被原子性的操作。也就是说读操作将获取与节点相关的所有数据，写操作也将替换掉节点的所有数据。另外，每一个节点都拥有自己的ACL(访问控制列表)，这个列表规定了用户的权限，即限定了特定用户对目标节点可以执行的操作。
  
- 节点类型

  **zookeeper中的节点有两种，分别为临时节点和永久节点。节点的类型在创建时即被确定，并且不能改变**。
    - **临时节点**：*该节点的生命周期依赖于创建它们的会话。一旦会话(Session)结束，临时节点将被自动删除*，当然可以也可以手动删除。虽然每个临时的Znode都会绑定到一个客户端会话，但他们对所有的客户端还是可见的。另外，*zookeeper的临时节点不允许拥有子节点*。
    - **永久节点**：该节点的生命周期不依赖于会话，并且只有在客户端显示执行删除操作的时候，他们才能被删除。
    
- 顺序节点
  
  **当创建Znode的时候，用户可以请求在zookeeper的路径结尾添加一个递增的计数**。这个计数对于此节点的父节点来说是唯一的，它的格式为"%10d"(10位数字，没有数值的数位用0补充，例如"0000000001")。当计数值大于232-1时，计数器将溢出。
    
- 监视器
  
  客户端可以在节点上设置watch，我们称之为监视器。**当节点状态发生改变时(Znode的增、删、改)将会触发watch所对应的操作**。当watch被触发时，zookeeper将会向客户端发送且仅发送一条通知，因此watch只能被触发一次，这样可以减少网络流量。

### zookeeper中的时间

zookeeper有多种记录时间的形式，其中包含以下几个主要属性：

- Zxid
  
  致使zookeeper节点状态改变的每一个操作都将使节点接收到一个Zxid格式的时间戳，并且这个时间戳全局有序。也就是说，也就是说，**每个对节点的改变都将产生一个唯一的Zxid**。如果Zxid1的值小于Zxid2的值，那么Zxid1所对应的事件发生在Zxid2所对应的事件之前。实际 上，zookeeper的每个节点维护者三个Zxid值，为别为：cZxid、mZxid、pZxid。
    - cZxid：节点创建时间所对应的Zxid格式时间戳
    - mZxid：节点最近一次修改的时间所对应的Zxid格式时间戳
    - pZxid：子节点（或该节点）的最近一次创建/删除的时间所对应的Zxid格式时间戳
      
  实现中Zxid是一个64为的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个 新的epoch。低32位是个递增计数。

- 版本号

  对节点的每一个操作都将致使这个节点的版本号增加。每个节点维护着三个版本号，他们分别为：
  - version：节点数据版本号
  - cversion：子节点版本号
  - aversion：节点所拥有的ACL版本号
  
### zookeeper节点属性

| 属性 | 描述 | 
| --- | --- | 
| cZxid | 节点创建时间所对应的Zxid格式时间戳 | 
| mZxid | 节点最近一次修改的时间所对应的Zxid格式时间戳 | 
| pZxid | 子节点（或该节点）的最近一次创建/删除的时间所对应的Zxid格式时间戳 | 
| ctime | 节点创建时间 | 
| mtime | 节点最近修改时间 | 
| version | 节点数据版本号 | 
| cversion | 子节点版本号 | 
| aversion | 节点所拥有的ACL版本号 | 
| ephemeralOwner | 如果此节点为临时节点，这个值为会话ID，否则，值为0 | 
| dataLength | 节点数据长度 | 
| numChildren | 子节点数量 | 

###  zookeeper节点操作

| 操作 | 描述 | 
| --- | --- | 
| create | 创建Znode(父节点必须存在) | 
| delete | 删除Znode(没有子节点) | 
| exists | 测试Znode是否存在，并获取它的元数据 | 
| getACL/setACL | 为Znode获取/设置ACL | 
| getChildren | 获取Znode所有子节点列表 | 
| getData/setData | 获取/设置Znode相关数据 | 
| sync | 使客户端的Znode视图与zookeeper同步 | 

更新zookeeper操作是有限制的。delete或setData必须明确要更新的Znode的版本号，我们可以调用exists找到。如果版本号不匹配，更新将会失败。
更新zookeeper操作是非阻塞式的。因此客户端如果失去了一个更新(由于另一个进程在同时更新这个Znode)，他可以在不阻塞其他进程执行的情况下，选择重新尝试或进行其他操作。

## watch触发器

**zookeeper可以为所有的读操作设置watch，这些读操作包括：`exists()`、`getChildren()`及`getData()`**。watch事件是**一次性的触发器**，当watch的对象状态发生改变时，将会触发此对象上watch所对应的事件。watch事件将被异步地发送给客户端，并且zookeeper为watch机制提供了有序的一致性保证。理论上，客户端接收watch事件的时间要快于其看到watch对象状态变化的时间。

### watch类型

- **数据watch(data watches)**：getData和exists负责设置数据watch;
- **孩子watch(child watches)**：getChildren负责设置孩子watch。

### watch注册与处触发
![](../../../public/images/zookeeper/watch.png)

1. `exists`操作上的watch，在被监视的Znode创建、删除或数据更新时被触发。
2. `getData`操作上的watch，在被监视的Znode删除或数据更新时被触发。在被创建时不能被触发，因为只有Znode一定存在，`getData`操作才会成功。
3. `getChildren`操作上的watch，在被监视的Znode的子节点创建或删除，或是这个Znode自身被删除时被触发。可以通过查看watch事件类型来区分是Znode，还是他的子节点被删除：`NodeDelete`表示Znode被删除，`NodeDeletedChanged`表示子节点被删除。

## zookeeper的简单操作

下载zookeeper：[https://downloads.apache.org/zookeeper/zookeeper-3.5.9/apache-zookeeper-3.5.9-bin.tar.gz](https://downloads.apache.org/zookeeper/zookeeper-3.5.9/apache-zookeeper-3.5.9-bin.tar.gz)

### shell操作

进入zookeeper目录，在conf目录下，新建一个名为`zoo.cfg`的文件，其中内容如下：

```cfg
# 服务器与客户端之间交互的基本时间单元（ms） 
tickTime=2000   
# zookeeper所能接受的客户端数量 
initLimit=10  
# 服务器和客户端之间请求和应答之间的时间间隔 
syncLimit=5
# zookeeper中使用的基本时间单位, 毫秒值.
tickTime=2000
# 数据目录. 可以是任意目录.
dataDir=/tmp/zookeeper/data
# log目录, 同样可以是任意目录. 如果没有设置该参数, 将使用和#dataDir相同的设置.
dataLogDir=/tmp/zookeeper/log
# t监听client连接的端口号.
clientPort=2181
```

启动zookeeper服务：`./bin/zkServer.sh start`;
在启动Zookeeper服务之后，输入`./bin/zkCli.sh -server localhost:2181`，连接到Zookeeper服务。

连接成功之后，系统会输出Zookeeper的相关环境及配置信息，并在屏幕输出“welcome to Zookeeper！”等信息。

1.  `ls path` : 列出指定路径节点下包含的子节点信息，例如 `ls /`
    ![](../../../public/images/zookeeper/ls-path.png)

2. `create path data`: 创建一个路径节点。例如 `create /zk hello`
   ![](../../../public/images/zookeeper/create-path-data.png)

3. `get path`: 获取指定路径节点的信息
   ![](../../../public/images/zookeeper/get-path.png)

4. `set path data`: 给指定路径节点设置数据
   ![](../../../public/images/zookeeper/set-path-data.png)

5. `delete path`: 删除指定路径节点
   ![](../../../public/images/zookeeper/delete-path.png)
   
### Zookeeper的API的简单使用

Zookeeper API共包含五个包，分别为：

- org.apache.zookeeper
- org.apache.zookeeper.data
- org.apache.zookeeper.server
- org.apache.zookeeper.server.quorum
- org.apache.zookeeper.server.upgrade

其中`org.apache.zookeeper`，包含`Zookeeper`类，他是我们编程时最常用的类文件。这个类是Zookeeper客户端的主要类文件。如果要使用Zookeeper服务，应用程序首先必须创建一个Zookeeper实例， 这时就需要使用此类。一旦客户端和Zookeeper服务建立起了连接，Zookeeper系统将会给次连接会话分配一个ID值，并且客户端将会周期性的 向服务器端发送心跳来维持会话连接。只要连接有效，客户端就可以使用`Zookeeper API`来做相应处理了。
![](../../../public/images/zookeeper/api.png)

## curator使用详解

**`Apache Curator`是一个比较完善的zookeeper客户端框架，通过封装的一套高级API，简化了ZooKeeper的操作，因此在实际应用中都是使用`Apache Curator`来操作zookeeper的**。通过查看官方文档，可以发现Curator主要解决了三类问题：

- 封装`ZooKeeper client`与`ZooKeeper server`之间的连接处理。
- 提供了一套Fluent风格的操作API。
- 提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装。

### 引入依赖

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

### 创建会话

#### 使用静态工程方法创建客户端

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

#### 使用Fluent风格的Api创建客户端

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

#### 创建包含隔离命名空间的客户端

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

### 数据节点操作

#### 创建数据节点

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

#### 删除数据节点

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

#### 读取数据节点数据

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

#### 更新数据节点数据

**更新一个节点的数据内容**

```java
client.setData().forPath("/path","data".getBytes());
```

**更新一个节点的数据内容，强制指定版本进行更新**

```java
client.setData().withVersion(10086).forPath("/path","data".getBytes());
```

#### 检查节点是否存在

```java
Stat stat = client.checkExists().forPath("/path");
```

注意：该方法返回一个Stat实例，不存在，则返回null。


#### 获取某个节点的所有子节点路径

```java
List<String> list = client.getChildren().forPath("/path");
```

该方法的返回值为List<String>,获得ZNode的子节点Path列表。 

#### 事务

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

### 缓存

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

#### Node Cache

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

#### Tree Cache

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

### Leader选举

在分布式计算中，`leader elections`是很重要的一个功能。这个选举过程是这样子的： 指派一个进程作为组织者，将任务分发给各节点。 在任务开始前， 哪个节点都不知道谁是leader(领导者)或者coordinator(协调者). 当选举算法开始执行后， 每个节点最终会得到一个唯一的节点作为任务leader. 除此之外，选举还经常会发生在leader意外宕机的情况下，新的leader要被选举出来。

**在zookeeper集群中，leader负责写操作，然后通过Zab协议实现follower的同步，leader或者follower都可以处理读操作**。
Curator有两种leader选举的recipe,分别是`LeaderSelector`和`LeaderLatch`。

- `LeaderSelector`：所有存活的客户端不间断的轮流做Leader。
- `LeaderLatch`：一旦选举出Leader，除非有客户端挂掉重新触发选举，否则不会交出领导权。

#### LeaderLatch

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

#### LeaderSelector

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

### 分布式锁

1. 推荐使用`ConnectionStateListener`监控连接的状态，因为当连接LOST时你不再拥有锁。
2. 分布式的锁全局同步，这意味着任何一个时间点不会有两个客户端都拥有相同的锁。

#### 可重入共享锁—Shared Reentrant Lock

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

#### 不可重入共享锁—Shared Lock

这个锁和上面的`InterProcessMutex`相比，就是少了`Reentrant`的功能，也就意味着它不能在同一客户端中重入。这个类是`InterProcessSemaphoreMutex`,使用方法和`InterProcessMutex`类似

> 源码见`InterProcessSemaphoreMutexDemo`类。

运行后发现，有且只有一个client成功获取第一个锁(第一个`acquire()`方法返回true)，然后它自己阻塞在第二个`acquire()`方法，获取第二个锁超时；其他所有的客户端都阻塞在第一个`acquire()`方法超时并且抛出异常。
这样也就验证了`InterProcessSemaphoreMutex`实现的锁是不可重入的。


