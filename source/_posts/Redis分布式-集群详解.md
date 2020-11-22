---
title: Redis分布式-集群详解
tags: redis
categories: 数据库
abbrlink: 2663121991
date: 2020-11-08 14:29:05
---

**Redis集群通过分片来进行数据共享，并提供复制和故障转移功能**。本文将对集群的节点、槽指派、命令执行、重新分片、转向、故障转移等各个方面进行介绍。

<!--more-->

> 本文主要内容参考自《Redis设计与实现》

## 节点

**一个Redis集群通常由多个节点(node)组成，在刚开始的时候，每个节点都是相互独立的，它们都处于一个只包含自己的集群当中**。可以通过`CLUSTER MEET`命令来连接各个节点，从而构建一个包含多节点集群。命令格式如下：

```text
CLUSTER MEET <ip> <port>
```

**当一个节点执行`CLUSTER MEET`命令，就可以将对应`ip`和`port`的节点加入到自己当前所在的集群中**（首先会进行握手，握手成功后才添加进来）。假设现在有三个节点，ip都是`127.0.0.1`，端口分别是`7000`、`7001`、`7002`。当在`7000`端口上的节点执行`CLUSTER MEET 127.0.0.1 7001`，就会将`7001`节点加入到`7000`节点所在的集群中。在`7000`端口上的节点再执行`CLUSTER MEET 127.0.0.1 7002`时，就会将`7002`节点也加入到`7000`节点所在的集群中。至此，这三个节点就组成了一个集群。
![cluster-node](https://chentianming11.github.io/images/redis/cluster-node.png)

### 启动节点

**一个节点就是一个运行在集群模式下的Redis服务器，Redis服务器在启动的时候根据`cluster-enabled`配置项的值来决定是否开启集群模式**。如果是`yes`，开启集群模式成为一个节点；否则，开启单机模式成为一个普通服务器。

节点会继续使用`redisServer`结构保存服务器状态，使用`redisClient`结构保存客户端状态，至于那些只在集群模式下才会用到的数据，节点将它们保存在`cluster.h/clusterNode`结构，`cluster.h/clusterLink`结构和`cluster.h/clusterState`结构里面。

### 集群数据结构

`clusterNode`结构保存了节点的当前状态，比如节点的创建时间、节点的名字、节点当前的配置纪元、节点的IP和端口等等。**每个节点都会使用一个`clusterNode`结构记录自己的状态，并且会为集群中所有其他节点都创建一个相应的`clusterNode`结构**。

```c
struct clusterNode {

    // 创建节点的时间
    mstime_t ctime;

    // 节点的名字，由40个十六进制字符组成
    char name[REDIS_CLUSTER_NAMELEN];

    // 节点标识
    // 使用不同的标识值记录节点的角色（比如主节点或者从节点）
    // 以及节点目前所处的状态（比如在线或者下线）
    int flags;

    // 节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;

    // 节点的IP地址
    char ip[REDIS_IP_STR_LEN];

    // 节点的端口号
    int port;

    // 保存连接节点所需的有关信息
    clusterLink *link;

    // ...

};
```

clusterNode结构的link属性是一个clusterLink结构，该结构保存了连接节点所需的有关信息，比如套接字描述符，输入缓存区和输出缓存区：

```c
typedef struct clusterLink {
    // 连接的创建时间
    mstime_t ctime;

    // TCP套接字描述符
    int fd;

    // 输出缓存区，保存着等待发送给其它节点的消息
    sds sndbuf;

    // 输入缓存区，保存着从其他节点接收到的消息
    sds rcvbuf;

    // 与这个连接相关联的节点，如果没有的话为NULL
    struct clusterNode *node;
} clusterLink;
```

最后，每个节点都保存着一个`RedisState`结构，记录了在当前节点视角下，集群目前所处的状态。例如集群是在线还是下线，集群中包含多少个节点，集群当前的配置纪元等。

```c
typedef struct clusterState {
    // 指向当前节点的指针
    clusterNode *myself;

    // 集群当前的配置纪元，用于实现故障转移
    unit64_t currentEpoch;

    // 集群当前的状态：是在线还是下线
    int state;

    // 集群中至少处理着一个槽的节点数量
    int size;

    // 集群节点名单
    dict *nodes;

    // ...
} clusterState;
```

以前面介绍的7000、7001和7002为例，下图展示了节点7000创建的`clusterState`结构，这个结构从7000这个节点角度记录了集群以及集群包含的三个节点的当前状态。
![cluster-state](https://chentianming11.github.io/images/redis/cluster-state.png)

- `currentEpoch`的属性值为0，表示集群当前的配置纪元为0
- `size`属性值为0，表示集群目前没有任何节点在处理槽，因此`state`为`REDIS_CLUSTER_FAIL`，表示节点处于下线状态。
- `nodes`属性记录了集群目前包含的三个节点
- 三个节点的`flags`属性都是`REDIS_NODE_MASTER`，表示三个节点都是主节点。

## 槽指派

**Redis集群通过分片的方式来保存数据库中的键值对：集群的数据库被分为16384个槽(slot)，数据库中的每个键都属于这16384个槽中的一个，集群中的每个节点可以处理0个或者最多16384个槽**。

**当数据库中的16384个槽都有节点在处理时，集群处于上线状态(ok)，否则处理下线状态(fail)**。

在上一节中，我们将7000、7001和7002三个节点连接到同一集群，不过此时，这个集群仍然处于下线状态，因为集群中的三个节点都没有在处理任何槽。通过向节点发送`CLUSTER ADDSLOTS`命令，可以将一个或者多个槽指派(assgin)给节点负责。

```text
CLUSTER ADDSLOTS <slot> [slot ...]
```

比如，我们可以将`0-5000`指派给7000负责，`5001-100000`指派给7002负责，`10001-16383`指派给7002负责。当完成指派之后，整个集群就会处于上线状态。

### 记录节点的槽指派信息

`clusterNode`结构的`slots`属性和`numslot`属性记录了节点负责处理的哪些槽：

```c
struct clusterNode {
    // ...
    unsigned char slots[16384/8];

    int numslots;
}
```

`slots`属性是一个二进制数组，长度为`16384/8=2048`个字节，共`16384`位，刚好对应`16384`个槽。如果二进制位是1，那么该位置的槽就由当前节点处理。`numslot`属性记录了`slots`二进制数组中值为1的个数，也就是当前节点处理的槽的数量。

### 传播节点的槽指派信息

一个节点除了会将自己负责处理的槽记录在`clusterNode`结构的`slots`属性和`numslot`属性中，还会将其发送给集群中的其它节点。这样的话，**集群中的每个节点都会知道数据库中的16384个槽被指派给了哪些节点**。

### 记录集群所有的槽指派信息

通过前面的记录节点的槽指派信息和传播节点的槽指派信息，实际上集群中的每个节点都会知道数据库中的16384个槽被指派给了哪些节点。但是具体要知道某个槽对应的节点，上述的数据结构的时间复杂度是`O(N)`。为了提高效率，`clusterState`结构中的`slots`数组记录了所有16384个槽的指派信息。

```c
typedef struct clusterState {

    clusterNode *slots[16384];
} clusterState;
```

`slots`数组包含16384个项，每个项是指向`clusterNode`的指针。这样的话，要知道某个槽对应的节点，时间复杂度就是`O(1)`。如果`slots[i]`指向`NULL`，就表示这个槽位没有被指派。

## 在集群中执行命令

在对数据16384个槽进行指派之后，集群就会进入上线状态。这时客户端节能向集群发送命令了。当客户端向节点发送与数据库键有关命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，然后检查这个槽是否指派给了自己：

- 如果指派给了自己，那么当前节点直接执行这个命令。
- 如果指派给了其它节点，那么当前节点会向客户端返回`MOVED`错误(包含这个键所属的节点)，客户端再转向正确的节点发送数据命令。

![cluster-command](https://chentianming11.github.io/images/redis/cluster-command.png)

### 计算键属于哪个槽

节点使用一下算法来计算给定键key属于哪个槽：

```c
def slot_number(key):
    return CRC16(key) & 16383
```

使用`CLUSTER KEYSLOT <key>`命令可以查看给定键属于哪个槽。

### 命令处理

当计算出键所属的槽i之后，接下来就会到`clusterState.slots[i]`找到这个槽对应的节点。如果该节点就是当前节点，那么直接执行命令，否则，返回`MOVED`错误，`MOVED`错误格式为`MOVED <slot> <ip>:<port>`。这样，客户端在收到`MOVED`错误之后，就能将命令转向给正确的节点执行了。

### 节点数据库实现

节点和单机服务器在数据库方面有一个区别，**节点只能使用0号数据库**。另外，除了将键值对保存在数据库之外，节点还会用`clusterState`结构中的`slots_to_keys`跳跃表来保存槽和键的关系。跳跃表的`score`是槽号，`value`是数据库键。**通过`slots_to_keys`跳跃表，节点可以很方便的对某个或者某些槽的数据库键进行批量操作**。

```c
typedef struct clusterState {

    // ...
    zskiplist *slots_to_keys;
    // ...
} clusterState;
```

## 重新分片

**`Redis`集群的重新分片操作可以将任意数量的已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点移动到目标节点**。

重新分片操作可以在线（online）进行，在重新分片的过程中，集群不需要下线，并且源节点和目标节点都可以继续处理请求。

比如，我们新启动一个端口为7003的服务器，将原本指派给7002的`15001-16383`的槽改指派给节点7003。

### 重新分片实现原理

Redis集群的重新分片操作是由Redis的集群管理软件`redis-trib`负责执行的。`redis-trib`对集群单个槽slot重新分片步骤如下：

1. `redis-trib`对目标节点发送`CLUSTER SETSLOT <slot> IMPORTING <source_id>`命令，让目标节点准备好从源节点导入槽slot的键值对。
2. `redis-trib`对源节点发送`CLUSTER SETSLOT <slot> MIGRATING <target_id>`命令，让源节点准备好将属于槽slot的键值对迁移(migrate)至目标节点。
3. `redis-trib`向源节点发送`CLUSTER GETKEYSINSLOT <slot> <count>`命令，获取最多count个属于槽slot的键名。
4. 对于步骤三的每个键名，`redis-trib`都向源节点发送一个`MIGRATE <target_id> <target_port> <key_name> 0 <time_out>`命令，将该键原子地从源节点迁移到目标节点。
5. 重新执行步骤三和四，直到槽slot的键值对全部迁移至目标节点。
6. `redis-trib`向集群任意一个节点发送`CLUSTER SETSLOT <slot> NODE <target_id>`命令，将槽slot指派给目标节点。并且指派信息会通过消息发送到整个集群。
![cluster-migrate](https://chentianming11.github.io/images/redis/cluster-migrate.png)

如果重新分片涉及多个槽，就对每个槽执行上述步骤。对slot槽进行重新分片过程如下：
![cluster-sharding](https://chentianming11.github.io/images/redis/cluster-sharding.png)

### ASK错误

在重新分片期间，源节点向目标节点迁移一个槽的过程中，可能出现术语被迁移槽的一部分键值对保存在源节点里面，而另一部分键值对则保存在目标节点里面。当客户端向源节点发送数据库键相关的命令，并且数据库键恰好就属于正在被迁移的槽时。**源节点首先会在自己的数据库里面查找该数据库键，如果找到则直接在源节点执行命令，否则，源节点返回一个`ASK`错误，指引客户端转向目标节点**。
![cluster-ask](https://chentianming11.github.io/images/redis/cluster-ask.png)

## 复制与故障转移

**`Redis`集群中的节点分为主节点和从节点，主节点用于处理槽，从节点用于复制某个主节点，并且在主节点下线时代替主节点继续进行客户端命令**。举个栗子，假设有7000、7001、7002、7003四个主节点，其中还有7004和7005作为70000的从节点。当7000节点下线时，会从7004和7005中选举出一个（假设选中了7004）作为代替7000的主节点。当7000重新上线时，7000会作为7004的从节点。
![cluster-failover](https://chentianming11.github.io/images/redis/cluster-failover.png)

> 主节点用双圆环表示。

### 设置从节点

向节点发送`CLUSTER REPLICATE <node_id>`命令，可以让该节点成为`node_id`的从节点。

### 故障检测

集群中每个节点都会定期向其他节点发送`PING`消息，以此来检测对方是否在线。如果对方节点在规定时间内没有返回`PONG`消息，那么发送方节点就会将对应的节点标记为疑似下线状态(PFIAL)。**如果一个集群内，半数以上主节点都将某个主节点x标记为疑似下线状态，那么这个节点x就会被标记为已下线状态(FAIL)**。

> 关键词：心跳检测+过半原则

### 故障转移

当从节点发现自己复制的主节点进入已下线状态时，从节点就开始对下线主节点进行故障转移，以下是故障转移的执行步骤。

1. 基于`Raft`算法，选举出一个从节点。
2. 被选中的从节点执行`SLAVEOF no one`，成为主节点。
3. 新的主节点会撤销已下线主节点的槽指派，并将这些槽全部指派给自己。
4. 新的主节点向集群广播一条`PONG`消息，让集群中其它节点知道新的主节点已经代替了已下线主节点。
5. 新的主节点开始接收和处理自己负责的槽有关的命令请求，故障转移完成。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~