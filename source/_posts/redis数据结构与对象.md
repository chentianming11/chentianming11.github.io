---
title: 一文读懂Redis常见对象类型的底层数据结构
tags: redis
categories: 数据库
abbrlink: 2583560655
date: 2020-08-30 13:25:45
---

`Redis`是一个基于内存中的数据结构存储系统，可以用作数据库、缓存和消息中间件。`Redis`支持五种常见对象类型：字符串(`String`)、哈希(`Hash`）、列表(`List`)、集合(`Set`)以及有序集合(`Zset`)，我们在日常工作中也会经常使用它们。知其然，更要知其所以然，本文将会带你读懂这五种常见对象类型的底层数据结构。

<!--more-->

> 本文主要内容参考自《Redis设计与实现》

## 对象类型和编码

**`Redis`使用对象来存储键和值的，在`Redis`中，每个对象都由`redisObject`结构表示**。`redisObject`结构主要包含三个属性：`type`、`encoding`和`ptr`。

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 底层数据结构的指针
    void *ptr;
} robj;
```

**其中`type`属性记录了对象的类型，对于`Redis`来说，键对象总是字符串类型，值对象可以是任意支持的类型**。因此，当我们说`Redis`键采用哪种对象类型的时候，指的是对应的值采用哪种对象类型。

| 类型常量 | 对象类型名称 |
| --- | --- |
| REDIS_STRING | 字符串对象 |
| REDIS_LIST | 列表对象 |
| REDIS_HASH | 哈希对象 |
| REDIS_SET | 集合对象 |
| REDIS_ZSET | 有序集合对象 |

**`*ptr`属性指向了对象的底层数据结构，而这些数据结构由`encoding`属性决定**。

| 编码常量 | 编码对应的底层数据结构 |
| --- | --- |
| REDIS_ENCODING_INT | long类型的整数 |
| REDIS_ENCODING_EMBSTR | emstr编码的简单动态字符串 |
| REDIS_ENCODING_RAW | 简单动态字符串 |
| REDIS_ENCODING_HT | 字典 |
| REDIS_ENCODING_LINKEDLIST | 双端链表 |
| REDIS_ENCODING_ZIPLIST | 压缩列表 |
| REDIS_ENCODING_INTSET | 整数集合 |
| REDIS_ENCODING_SKIPLIST | 跳跃表和字典 |

**之所以由`encoding`属性来决定对象的底层数据结构，是为了实现同一对象类型，支持不同的底层实现**。这样就能在不同场景下，使用不同的底层数据结构，进而极大提升`Redis`的灵活性和效率。

> 底层数据结构后面会详细讲解，这里简单看一下即可。

## 字符串对象

字符串是我们日常工作中用得最多的对象类型，它对应的编码可以是`int`、`raw`和`embstr`。**字符串对象相关命令可参考：[Redis命令-Strings](http://www.redis.cn/commands.html#string)**。

如果一个字符串对象保存的是不超过`long`类型的整数值，此时编码类型即为`int`，其底层数据结构直接就是`long`类型。例如执行`set number 10086`，就会创建`int`编码的字符串对象作为`number`键的值。
![redis-encoding-int](https://chentianming11.github.io/images/redis/redis-encoding-int.png)

如果字符串对象保存的是一个长度大于39字节的字符串，此时编码类型即为`raw`，其底层数据结构是简单动态字符串(`SDS`)；如果长度小于等于39个字节，编码类型则为`embstr`，底层数据结构就是`embstr`编码`SDS`。下面，我们详细理解下什么是简单动态字符串。

### 简单动态字符串

#### SDS定义

在`Redis`中，使用`sdshdr`数据结构表示`SDS`：

```c
struct sdshdr {
    // 字符串长度
    int len;
    // buf数组中未使用的字节数
    int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```

**`SDS`遵循了C字符串以空字符结尾的惯例，保存空字符的1字节不会计算在`len`属性里面**。例如，`Redis`这个字符串在`SDS`里面的数据可能是如下形式：
![sdshdr](https://chentianming11.github.io/images/redis/sdshdr.png)

#### SDS与C字符串的区别

C语言使用长度为`N+1`的字符数组来表示长度为`N`的字符串，并且字符串的最后一个元素是空字符`\0`。`Redis`采用`SDS`相对于C字符串有如下几个优势：

1. 常数复杂度获取字符串长度
2. 杜绝缓冲区溢出
3. 减少修改字符串时带来的内存重分配次数
4. 二进制安全

##### 常数复杂度获取字符串长度

因为C字符串并不记录自身的长度信息，所以为了获取字符串的长度，必须遍历整个字符串，时间复杂度是`O(N)`；而`SDS`使用`len`属性记录了字符串的长度，因此获取`SDS`字符串长度的时间复杂度是`O(1)`。

##### 杜绝缓冲区溢出

**C字符串不记录自身长度带来的另一个问题是很容易造成缓存区溢出**。比如使用字符串拼接函数(`stract`)的时候，很容易覆盖掉字符数组原有的数据。与C字符串不同，**`SDS`的空间分配策略完全杜绝了发生缓存区溢出的可能性**。当`SDS`进行字符串扩充时，首先会检查当前的字节数组的长度是否足够，如果不够的话，会先进行自动扩容，然后再进行字符串操作。

##### 减少修改字符串时带来的内存重分配次数

因为C字符串的长度和底层数据是紧密关联的，所以每次增长或者缩短一个字符串，程序都要对这个数组进行一次内存重分配：

- 如果是增长字符串操作，需要先通过内存重分配来扩展底层数组空间大小，不这么做就导致缓存区溢出。
- 如果是缩短字符串操作，需要先通过内存重分配来来回收不再使用的空间，不这么做就导致内存泄漏。

因为内存重分配涉及复杂的算法，并且可能需要执行系统调用，所以通常是个比较耗时的操作。**对于`Redis`来说，字符串修改是一个十分频繁的操作，如果每次都像C字符串那样进行内存重分配，对性能影响太大了，显然是无法接受的**。

`SDS`通过空闲空间解除了字符串长度和底层数据之间的关联。在`SDS`中，数组中可以包含未使用的字节，这些字节数量由`free`属性记录。**通过空闲空间，`SDS`实现了空间预分配和惰性空间释放两种优化策略**。

1. **空间预分配**
   **空间预分配是用于优化`SDS`字符串增长操作的，简单来说就是当字节数组空间不足触发重分配的时候，总是会预留一部分空闲空间**。这样的话，就能减少连续执行字符串增长操作时的内存重分配次数。有两种预分配的策略：
   1. `len`小于`1MB`时：每次重分配时会多分配同样大小的空闲空间；
   2. `len`大于等于`1MB`时：每次重分配时会多分配1MB大小的空闲空间。
2. **惰性空间释放**
   惰性空间释放是用于优化`SDS`字符串缩短操作的，简单来说就是当字符串缩短时，并不立即使用内存重分配来回收多出来的字节，而是用free属性记录，等待将来使用。`SDS`也提供直接释放未使用空间的`API`，在需要的时候，也能真正的释放掉多余的空间。

##### 二进制安全

C字符串中的字符必须符合某种编码，并且除了字符串末尾之外，其它位置不允许出现空字符，这些限制使得C字符串只能保存文本数据。但是对于`Redis`来说，不仅仅需要保存文本，还要支持保存二进制数据。为了实现这一目标，`SDS`的`API`全部做到了二进制安全(`binary-safe`)。

### raw和embstr编码的SDS区别

我们在前面讲过，长度大于39字节的字符串，编码类型为`raw`，底层数据结构是简单动态字符串(`SDS`)。这个很好理解，比如当我们执行`set story "Long, long, long ago there lived a king ..."`(长度大于39)之后，`Redis`就会创建一个`raw`编码的`String`对象。数据结构如下：
![string-raw](https://chentianming11.github.io/images/redis/string-raw.png)

长度小于等于39个字节的字符串，编码类型为`embstr`，底层数据结构则是`embstr`编码`SDS`。`embstr`编码是专门用来保存短字符串的，它和`raw`编码最大的不同在于：**`raw`编码会调用两次内存分配分别创建`redisObject`结构和`sdshdr`结构，而`embstr`编码则是只调用一次内存分配，在一块连续的空间上同时包含`redisObject`结构和`sdshdr`结构**。
![embstr](https://chentianming11.github.io/images/redis/embstr.png)

### 编码转换

`int`编码和`embstr`编码的字符串对象在条件满足的情况下会自动转换为`raw`编码的字符串对象。
对于`int`编码来说，当我们修改这个字符串为不再是整数值的时候，此时字符串对象的编码就会从`int`变为`raw`；对于`embstr`编码来说，只要我们修改了字符串的值，此时字符串对象的编码就会从`embstr`变为`raw`。

> `embstr`编码的字符串对象可以认为是只读的，因为`Redis`为其编写任何修改程序。当我们要修改`embstr`编码字符串时，都是先将转换为`raw`编码，然后再进行修改。

## 列表对象

**列表对象的编码可以是`linkedlist`或者`ziplist`，对应的底层数据结构是链表和压缩列表**。列表对象相关命令可参考：[Redis命令-List](http://www.redis.cn/commands.html#list)。
默认情况下，当列表对象保存的所有字符串元素的长度都小于64字节，且元素个数小于512个时，列表对象采用的是`ziplist`编码，否则使用`linkedlist`编码。

> 可以通过配置文件修改该上限值。

### 链表

链表是一种非常常见的数据结构，提供了高效的节点重排能力以及顺序性的节点访问方式。在`Redis`中，每个链表节点使用`listNode`结构表示：

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode
```

多个`listNode`通过`prev`和`next`指针组成双端链表，如下图所示：
![listnode](https://chentianming11.github.io/images/redis/listnode.png)

为了操作起来比较方便，`Redis`使用了`list`结构持有链表。

```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表包含的节点数量
    unsigned long len;
    // 节点复制函数
    void *(*dup)(void *ptr);
    // 节点释放函数
    void (*free)(void *ptr);
    // 节点对比函数
    int (*match)(void *ptr, void *key);
} list;
```

list结构为链表提供了表头指针`head`、表尾指针`tail`，以及链表长度计数器`len`，而`dup`、`free`和`match`成员则是实现多态链表所需类型的特定函数。
![list](https://chentianming11.github.io/images/redis/list.png)

`Redis`链表实现的特征总结如下：

1. **双端**：链表节点带有`prev`和`next`指针，获取某个节点的前置节点和后置节点的复杂度都是`O(n)`。
2. **无环**：表头节点的`prev`指针和表尾节点的`next`指针都指向`NULL`，对链表的访问以`NULL`为终点。
3. **带表头指针和表尾指针**：通过`list`结构的`head`指针和`tail`指针，程序获取链表的表头节点和表尾节点的复杂度为`O(1)`。
4. **带链表长度计数器**：程序使用`list`结构的`len`属性来对`list`持有的节点进行计数，程序获取链表中节点数量的复杂度为`O(1)`。
5. **多态**：链表节点使用`void*`指针来保存节点值，可以保存各种不同类型的值。

### 压缩列表

压缩列表(`ziplist`)是列表键和哈希键的底层实现之一。压缩列表主要目的是为了节约内存，是由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值。
![ziplist](https://chentianming11.github.io/images/redis/ziplist.png)
如上图所示，压缩列表记录了各组成部分的类型、长度以及用途。

| 属性 | 类型 | 长度 | 用途 |
| --- | --- | --- | --- |
| zlbytes | uint_32_t | 4字节 | 记录整个压缩列表占用的内存字节数 |
| zltail | uint_32_t | 4字节 | 记录压缩列表表尾节点距离起始地址有多少字节，通过这个偏移量，程序无需遍历整个压缩列表就能确定表尾节点地址 |
| zlen | uint_16_t | 2字节 | 记录压缩列表包含的节点数量 |
| entryX | 列表节点 | 不定 | 压缩列表的各个节点，节点长度由保存的内容决定 |
| zlend | uint_8_t | 1字节 | 特殊值(`0xFFF`)，用于标记压缩列表末端 |

## 哈希对象

哈希对象的编码可以是`ziplist`或者`hashtable`。

### hash-ziplist

`ziplist`底层使用的是压缩列表实现，上文已经详细介绍了压缩列表的实现原理。每当有新的键值对要加入哈希对象时，先把保存了键的节点推入压缩列表表尾，然后再将保存了值的节点推入压缩列表表尾。比如，我们执行如下三条`HSET`命令：

```shell
HSET profile name "tom"
HSET profile age 25
HSET profile career "Programmer"
```

如果此时使用`ziplist`编码，那么该`Hash`对象在内存中的结构如下：
![hash_ziplist](https://chentianming11.github.io/images/redis/hash_ziplist.png)

### hash-hashtable

`hashtable`编码的哈希对象使用字典作为底层实现。字典是一种用于保存键值对的数据结构，`Redis`的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，每个哈希表节点保存的就是一个键值对。

#### 哈希表

`Redis`使用的哈希表由`dictht`结构定义：

```c
typedef struct dictht{
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size-1
    unsigned long sizemask;

    // 该哈希表已有节点数量
    unsigned long used;
} dictht
```

`table`属性是一个数组，数组中的每个元素都是一个指向`dictEntry`结构的指针，每个`dictEntry`结构保存着一个键值对。`size`属性记录了哈希表的大小，即`table`数组的大小。`used`属性记录了哈希表目前已有节点数量。`sizemask`总是等于`size-1`，这个值主要用于数组索引。比如下图展示了一个大小为4的空哈希表。
![dictht](https://chentianming11.github.io/images/redis/dictht.png)

#### 哈希表节点

哈希表节点使用`dictEntry`结构表示，每个`dictEntry`结构都保存着一个键值对：

```c
typedef struct dictEntry {
    // 键
    void *key;

    // 值
    union {
        void *val;
        unit64_t u64;
        nit64_t s64;
    } v;

    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

`key`属性保存着键值对中的键，而`v`属性则保存了键值对中的值。值可以是一个指针，一个`uint64_t`整数或者是`int64_t`整数。`next`属性指向了另一个`dictEntry`节点，在数组桶位相同的情况下，将多个`dictEntry`节点串联成一个链表，以此来解决键冲突问题。(链地址法)

#### 字典

`Redis`字典由`dict`结构表示：

```c
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    //rehash索引
    // 当rehash不在进行时，值为-1
    int rehashidx;
}
```

`ht`是大小为2，且每个元素都指向`dictht`哈希表。一般情况下，字典只会使用`ht[0]`哈希表，`ht[1]`哈希表只会在对`ht[0]`哈希表进行`rehash`时使用。`rehashidx`记录了`rehash`的进度，如果目前没有进行rehash，值为-1。
![dict](https://chentianming11.github.io/images/redis/dict.png)

#### rehash

为了使hash表的负载因子(`ht[0]).used`/`ht[0]).size`)维持在一个合理范围，当哈希表保存的元素过多或者过少时，程序需要对hash表进行相应的扩展和收缩。`rehash`（重新散列）操作就是用来完成hash表的扩展和收缩的。rehash的步骤如下：

1. 为`ht[1]`哈希表分配空间
   1. 如果是扩展操作，那么`ht[1]`的大小为第一个大于`ht[0].used*2`的2^n。比如`ht[0].used=5`，那么此时`ht[1]`的大小就为16。(大于10的第一个2^n的值是16)
   2. 如果是收缩操作，那么`ht[1]`的大小为第一个大于`ht[0].used`的2^n。比如`ht[0].used=5`，那么此时`ht[1]`的大小就为8。(大于5的第一个2^n的值是8)
2. 将保存在`ht[0]`中的所有键值对rehash到`ht[1]`中。
3. 迁移完成之后，释放掉`ht[0]`，并将现在的`ht[1]`设置为`ht[0]`，在`ht[1]`新创建一个空白哈希表，为下一次rehash做准备。

**哈希表的扩展和收缩时机**：

1. 当服务器没有执行`BGSAVE`或者`BGREWRITEAOF`命令时，负载因子大于等于1触发哈希表的扩展操作。
2. 当服务器在执行`BGSAVE`或者`BGREWRITEAOF`命令，负载因子大于等于5触发哈希表的扩展操作。
3. 当哈希表负载因子小于0.1，触发哈希表的收缩操作。

#### 渐进式rehash

前面讲过，扩展或者收缩需要将`ht[0]`里面的元素全部rehash到`ht[1]`中，如果`ht[0]`元素很多，显然一次性rehash成本会很大，从影响到`Redis`性能。为了解决上述问题，`Redis`使用了**渐进式rehash**技术，具体来说就是**分多次，渐进式地将`ht[0]`里面的元素慢慢地rehash到`ht[1]`中**。下面是**渐进式rehash**的详细步骤：

1. 为`ht[1]`分配空间。
2. 在字典中维持一个索引计数器变量`rehashidx`，并将它的值设置为0，表示rehash正式开始。
3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新时，除了会执行相应的操作之外，还会顺带将`ht[0]`在`rehashidx`索引位上的所有键值对rehash到`ht[1]`中，rehash完成之后，`rehashidx`值加1。
4. 随着字典操作的不断进行，最终会在啊某个时刻迁移完成，此时将`rehashidx`值置为-1，表示rehash结束。

**渐进式rehash一次迁移一个桶上所有的数据，设计上采用分而治之的思想，将原本集中式的操作分散到每个添加、删除、查找和更新操作上**，从而避免集中式rehash带来的庞大计算。

因为在渐进式rehash时，字典会同时使用`ht[0]`和`ht[1]`两张表，所以此时对字典的删除、查找和更新操作都可能会在两个哈希表进行。比如，如果要查找某个键时，先在`ht[0]`中查找，如果没找到，则继续到`ht[1]`中查找。

#### hash对象中的hashtable

```shell
HSET profile name "tom"
HSET profile age 25
HSET profile career "Programmer"
```

还是上述三条命令，保存数据到`Redis`的哈希对象中，如果采用`hashtable`编码保存的话，那么该`Hash`对象在内存中的结构如下：
![hash-ht](https://chentianming11.github.io/images/redis/hash-ht.png)

当哈希对象保存的所有键值对的键和值的字符串长度都小于64个字节，并且数量小于512个时，使用`ziplist`编码，否则使用`hashtable`编码。

> 可以通过配置文件修改该上限值。

## 集合对象

集合对象的编码可以是`intset`或者`hashtable`。当集合对象保存的元素都是整数，并且个数不超过512个时，使用`intset`编码，否则使用`hashtable`编码。

### set-intset

`intset`编码的集合对象底层使用整数集合实现。

整数集合(intset)是`Redis`用于保存整数值的集合抽象数据结构，它可以保存类型为`int16_t`、`int32_t`或者`int64_t`的整数值，并且保证集合中的数据不会重复。`Redis`使用`intset`结构表示一个整数集合。

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

`contents`数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数组项，各个项在数组中按值大小从小到大有序排列，并且数组中不包含重复项。虽然`contents`属性声明为`int8_t`类型的数组，但实际上，`contents`数组不保存任何`int8_t`类型的值，数组中真正保存的值类型取决于`encoding`。如果`encoding`属性值为`INTSET_ENC_INT16`，那么`contents`数组就是`int16_t`类型的数组，以此类推。

当新插入元素的类型比整数集合现有类型元素的类型大时，整数集合必须先升级，然后才能将新元素添加进来。这个过程分以下三步进行。

1. 根据新元素类型，扩展整数集合底层数组空间大小。
2. 将底层数组现有所有元素都转换为与新元素相同的类型，并且维持底层数组的有序性。
3. 将新元素添加到底层数组里面。

还有一点需要注意的是，整数集合不支持降级，一旦对数组进行了升级，编码就会一直保持升级后的状态。

举个栗子，当我们执行`SADD numbers 1 3 5`向集合对象插入数据时，该集合对象在内存的结构如下：
![set-intset](https://chentianming11.github.io/images/redis/set-intset.png)

### set-hashtable

`hashtable`编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象对应一个集合元素，字典的值都是`NULL`。当我们执行`SADD fruits "apple" "banana" "cherry"`向集合对象插入数据时，该集合对象在内存的结构如下：
![set-hash](https://chentianming11.github.io/images/redis/set-hash.png)

## 有序集合对象

有序集合的编码可以是`ziplist`或者`skiplist`。当有序集合保存的元素个数小于128个，且所有元素成员长度都小于64字节时，使用`ziplist`编码，否则，使用`skiplist`编码。

### zset-ziplist

**`ziplist`编码的有序集合使用压缩列表作为底层实现，每个集合元素使用两个紧挨着一起的两个压缩列表节点表示，第一个节点保存元素的成员(member)，第二个节点保存元素的分值(score)**。

**压缩列表内的集合元素按照分值从小到大排列**。如果我们执行`ZADD price 8.5 apple 5.0 banana 6.0 cherry`命令，向有序集合插入元素，该有序集合在内存中的结构如下：
![zset-ziplist](https://chentianming11.github.io/images/redis/zset-ziplist.png)

### zset-skiplist

`skiplist`编码的有序集合对象使用`zset`结构作为底层实现，一个`zset`结构同时包含一个字典和一个跳跃表。

```c
typedef struct zset {
    zskiplist *zs1;
    dict *dict;
}
```

继续介绍之前，我们先了解一下什么是跳跃表。

#### 跳跃表

跳跃表(skiplist)是一种有序的数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。`Redis`的跳跃表由`zskiplistNode`和`zskiplist`两个结构定义，`zskiplistNode`结构表示跳跃表节点，`zskiplist`保存跳跃表节点相关信息，比如节点的数量，以及指向表头和表尾节点的指针等。

##### 跳跃表节点 zskiplistNode

跳跃表节点`zskiplistNode`结构定义如下：

```c
typedef struct zskiplistNode {
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

下图是一个层高为5，包含4个跳跃表节点(1个表头节点和3个数据节点)组成的跳跃表：
![zskiplistNode](https://chentianming11.github.io/images/redis/zskiplistNode.png)

1. 层
   **每次创建一个新的跳跃表节点的时候，会根据幂次定律(越大的数出现的概率越低)随机生成一个`1-32`之间的值作为当前节点的"层高"**。每层元素都包含2个数据，前进指针和跨度。
    1. 前进指针
       每层都有一个指向表尾方向的前进指针，用于从表头向表尾方向访问节点。
    2. 跨度
       层的跨度用于记录两个节点之间的距离。
2. 后退指针(BW)
   节点的后退指针用于从表尾向表头方向访问节点，每个节点只有一个后退指针，所以每次只能后退一个节点。
3. 分值和成员
   节点的分值(score)是一个`double`类型的浮点数，跳跃表中所有节点都按分值从小到大排列。节点的成员(obj)是一个指针，指向一个字符串对象。在跳跃表中，各个节点保存的成员对象必须是唯一的，但是多个节点的分值确实可以相同。

**需要注意的是，表头节点不存储真实数据，并且层高固定为32，从表头节点第一个不为`NULL`最高层开始，就能实现快速查找**。

##### 跳跃表 zskiplist

实际上，仅靠多个跳跃表节点就可以组成一个跳跃表，但是`Redis`使用了`zskiplist`结构来持有这些节点，这样就能够更方便地对整个跳跃表进行操作。比如快速访问表头和表尾节点，获得跳跃表节点数量等等。`zskiplist`结构定义如下：

```c
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct skiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 最大层数
    int level;
} zskiplist;
```

下图是一个完整的跳跃表结构示例：
![zskiplist](https://chentianming11.github.io/images/redis/zskiplist.png)

#### 有序集合对象的`skiplist`实现

**前面讲过，`skiplist`编码的有序集合对象使用`zset`结构作为底层实现，一个`zset`结构同时包含一个字典和一个跳跃表**。

```c
typedef struct zset {
    zskiplist *zs1;
    dict *dict;
}
```

`zset`结构中的`zs1`跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素。通过跳跃表，可以对有序集合进行基于`score`的快速范围查找。`zset`结构中的`dict`字典为有序集合创建了从成员到分值的映射，字典的键保存了成员，字典的值保存了分值。通过字典，可以用`O(1)`复杂度查找给定成员的分值。

假如还是执行`ZADD price 8.5 apple 5.0 banana 6.0 cherry`命令向`zset`保存数据，如果采用`skiplist`编码方式的话，该有序集合在内存中的结构如下：
![zset-skiplist](https://chentianming11.github.io/images/redis/zset-skiplist.png)

## 总结

**总的来说，`Redis`底层数据结构主要包括简单动态字符串(SDS)、链表、字典、跳跃表、整数集合和压缩列表六种类型，并且基于这些基础数据结构实现了字符串对象、列表对象、哈希对象、集合对象以及有序集合对象五种常见的对象类型**。每一种对象类型都至少采用了2种数据编码，不同的编码使用的底层数据结构也不同。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~
