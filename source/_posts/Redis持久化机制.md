---
title: 深入理解Redis持久化机制-RDB、AOF实现详解
tags: redis
categories: 数据库
abbrlink: 4169530895
date: 2020-10-17 15:53:01
---

`Redis`是一个基于内存中的数据结构存储系统，它所有的数据都存储在内存中。如果发生断电或者宕机，内存中的数据就会丢失。为了防止数据丢失，`Redis`提供了两种持久化的方案，一种是`RDB(Redis DataBase)`，另一种是`AOF(Append Only File)`。

<!--more-->

> 本文主要内容参考自《Redis设计与实现》

## RDB持久化

`RDB`持久化指的是将某个时间点上的数据库状态保存到一个`RDB`文件中，在启动的时候，可以通过`RDB`文件将数据库状态还原回来。
![rdb](https://chentianming11.github.io/images/redis/rdb.png)

### RDB文件的创建和载入

有2个`Redis`命令可以生成`RDB`文件，一个是`SAVE`，另一个是`BGSAVE`。`SAVE`命令会阻塞`Redis`服务进程，直到`RDB`文件创建完成为止，在此期间，客户端所有命令请求都会被阻塞。而`BGSAVE`命令会派生出一个子进程，由子进程负责创建`RDB`文件，服务器进程继续处理命令请求。当然，在`BGSAVE`期间，服务器处理`SAVE`、`BGSAVE`和`BGREWRITEAOP`三个命令的方式会有所不同。具体来说就是，在`BGSAVE`期间，服务器会拒绝执行`SAVE`、`BGSAVE`命令（防止产出竞争）；对于`BGREWRITEAOP`命令，服务器会延迟到`BGSAVE`执行完成才真正开始执行。

创建`RDB`文件实际上由`rdb.c/rdbSave`函数完成，`SAVE`和`BGSAVE`命令会以不同的方式调用该函数，伪代码如下：

```python
def SAVE():
    # 创建RDB文件
    rdbSave()

def BGSAVE():
    # 创建子进程
    pid = fork()
    if pid == 0:
        # 子进程负责创建RDB文件
        rdbSave()
        # 完成之后向父进程发送信号
        signal_parent()
    elif pid > 0:
        # 父进程继续处理命令请求，并通过轮询等待子进程信号
        handle_request_and_wait_sifnal()
    else:
        # 处理出错
        handle_fork_error()
```

> 由`fork`创建的新进程被称为子进程（`child process`）。**该函数被调用一次，但返回两次**。两次返回的区别是子进程的返回值是0，而父进程的返回值则是新进程（子进程）的进程id。`fork`之后，操作系统会复制一个与父进程完全相同的子进程，虽说是父子关系，但是在操作系统看来，他们更像兄弟关系，这2个进程共享代码空间，但是数据空间是互相独立的，子进程数据空间中的内容是父进程的完整拷贝，指令指针也完全相同，子进程拥有父进程当前运行到的位置。具体可参考[fork出的子进程和父进程](https://blog.csdn.net/koudaidai/article/details/8014782)。

`RDB`文件载入是在`Redis`启动的时候自动执行的。在服务只开启`RDB`持久化时，只要在启动的时候检测到`RDB`文件，就会执行自动载入。在`RDB`文件载入期间，服务器会一直处理阻塞状态，直到载入完成为止。

### 自动间隔保存

除了手动执行`SAVE`或者`BGSAVE`命令来创建`RDB`文件，`Redis`还支持通过设置服务器配置的`save`选项，让服务器每隔一段时间自动执行`BGSAVE`命令。

用户可以设置多个保存条件，只要其中一个条件被满足，服务器就会执行`BGSAVE`命令。例如：

```text
save 900 1
save 300 10
save 60 10000
```

那么满足以下任意一个条件，`BGSAVE`命令就会被执行：

1. 服务器在900秒之内，对数据库进行了至少1次修改。
2. 服务器在300秒之内，对数据库进行了至少10次修改。
3. 服务器在60秒之内，对数据库进行了至少10000次修改。

### RDB文件结构

一个完整`RDB`文件包含的各个部分如下图所示：
![rdbfile](https://chentianming11.github.io/images/redis/rdbfile.png)

> 为了方便区分变量、数据和常量，本文用全大写单词表示常量，用全小写单词表示变量和数据。

| 数据部分 | 数据类型 | 长度 | 数据含义 |
| --- | --- | --- | --- |
| REDIS | 常量 | 5字节 | `RDB`文件标识，用来快速检查载入的文件是否是`RDB`文件 |
| db_version | 变量 | 4字节 | `RDB`文件版本号 |
| database | 数据 | 不定 | `Redis`各个非空数据库状态 |
| EOF | 常量 | 1字节 | 标志着`RDB`文件正文内容结束 |
| check_sum | 变量 | 8字节 | 校验和，根据前面4部分计算而来，用来检查`RDB`文件完整性 |

#### database部分

`database`部分保存了任意多个非空数据库状态，对于每一个非空数据库，`database`结构如下：
![rdb-database](https://chentianming11.github.io/images/redis/rdb-database.png)

| 数据部分 | 数据类型 | 长度 | 数据含义 |
| --- | --- | --- | --- |
| SELECT_DB | 常量 | 1字节 | 数据库开头标识 |
| db_number | 变量 | 1-5个字节 | 数据库开头号码 |
| key_value_pairs | 数据 | 不定 | 数据库所有键值对数据 |

#### key_value_pairs部分

`key_value_pairs`部分保存了一个数据所有的键值对数据，其中不带过期时间的键值对有`TYPE`、`key`、`value`三部分组成，带过期时间的话，会在前面多出`EXPIRETIME_MS`和`ms`两部分。
![rdb-kv](https://chentianming11.github.io/images/redis/rdb-kv.png)

| 数据部分 | 数据类型 | 长度 | 数据含义 |
| --- | --- | --- | --- |
| EXPIRETIME_MS | 常量 | 1字节 | 键值对过期时间标识 |
| ms | 变量 | 8字节 | 键值对的过期时间（毫秒） |
| TYPE | 常量 | 1字节 | 数据库值的类型 |
| key | 变量 | 不定 | 数据库键，永远是字符串对象 |
| value | 变量 | 不定 | 数据库值，根据`TYPE`不用，`value`保存结构也不同 |

> 每个`value`都保存着一个值对象，每个值对象的类型由`TYPE`字段记录。根据`TYPE`类型不同，`value`部分的结构、长度也会有所不同。这块思想上跟内存中底层数据结构类似，这里就不展开细讲了。

至此，一个`RDB`文件完整的部分就出来了，如下所示：
![rdb-all](https://chentianming11.github.io/images/redis/rdb-all.png)

## AOF持久化

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~
