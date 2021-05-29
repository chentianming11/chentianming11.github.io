---
title: zookeeper基本介绍及使用
tags: zookeeper
categories: zookeeper
abbrlink: 4128331822
date: 2021-05-04 10:51:29
---

**zookeeper是一种为分布式应用所设计的高可用、高性能且一致的开源协调服务，主要可用来实现分布式锁、配置维护、分布式消息队列、分布式通知/协调等**。

<!--more-->

zookeeper设计了一种全新的数据结构(`Znode`)，并且在`Znode`上定义了一些原语（关于该数据结构的一些操作）。因为zookeeper是工作在一个分布式的环境下，所以还需要一个通知机制(**watch机制**)来将消息通过网络发送给分布式应用程序。总结一下，**zookeeper所提供的服务主要是通过：Znode数据结构+原语+watch机制，三个部分来实现的**，那么接下来就从这三个方面来简单介绍一下zookeeper。


源码：[https://github.com/chentianming11/zookeeper-demo](https://github.com/chentianming11/zookeeper-demo)

## zookeeper数据模型

### Znode

zookeeper拥有一个层次的命名空间，这个和标准的文件系统非常相似，如下图所示。
![](https://chentianming11.github.io/images/zookeeper/znode.png)
![](https://chentianming11.github.io/images/zookeeper/znode-file.png)

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
![](https://chentianming11.github.io/images/zookeeper/watch.png)

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
    ![](https://chentianming11.github.io/images/zookeeper/ls-path.png)

2. `create path data`: 创建一个路径节点。例如 `create /zk hello`
   ![](https://chentianming11.github.io/images/zookeeper/create-path-data.png)

3. `get path`: 获取指定路径节点的信息
   ![](https://chentianming11.github.io/images/zookeeper/get-path.png)

4. `set path data`: 给指定路径节点设置数据
   ![](https://chentianming11.github.io/images/zookeeper/set-path-data.png)

5. `delete path`: 删除指定路径节点
   ![](https://chentianming11.github.io/images/zookeeper/delete-path.png)
   
### Zookeeper的API的简单使用

Zookeeper API共包含五个包，分别为：

- org.apache.zookeeper
- org.apache.zookeeper.data
- org.apache.zookeeper.server
- org.apache.zookeeper.server.quorum
- org.apache.zookeeper.server.upgrade

其中`org.apache.zookeeper`，包含`Zookeeper`类，他是我们编程时最常用的类文件。这个类是Zookeeper客户端的主要类文件。如果要使用Zookeeper服务，应用程序首先必须创建一个Zookeeper实例， 这时就需要使用此类。一旦客户端和Zookeeper服务建立起了连接，Zookeeper系统将会给次连接会话分配一个ID值，并且客户端将会周期性的 向服务器端发送心跳来维持会话连接。只要连接有效，客户端就可以使用`Zookeeper API`来做相应处理了。
![](https://chentianming11.github.io/images/zookeeper/api.png)

