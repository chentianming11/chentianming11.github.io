---
title: 深入理解mysql-InnoDB数据结构
tags:
  - mysql
categories:
  - 数据库
abbrlink: 786374041
date: 2020-06-10 21:45:41
---

InnoDB一个支持事务安全的存储引擎，同时也是mysql的默认存储引擎。本文主要从数据结构的角度，详细介绍InnoDB行记录格式和数据页的实现原理，帮助读者更加具象地理解InnoDB存储引擎。
<!--more-->

## InnoDB简介

大家都知道mysql中数据是存储在物理磁盘上的，而真正的数据处理又是在内存中执行的。由于磁盘的读写速度非常慢，如果每次操作都对磁盘进行频繁读写的话，那么性能一定非常差。为了上述问题，**InnoDB将数据划分为若干页，以页作为磁盘与内存交互的基本单位，一般页的大小为16KB**。这样的话，一次性至少读取1页数据到内存中或者将1页数据写入磁盘。通过减少内存与磁盘的交互次数，从而提升性能。

其实，这本质上就是一种典型的缓存设计思想，一般缓存的设计基本都是从`时间维度`或者`空间维度`进行考量的：

1. `时间维度`：如果一条数据正在在被使用，那么在接下来一段时间内大概率还会再被使用。可以认为`热点数据缓存`都属于这种思路的实现。
2. `空间维度`：如果一条数据正在在被使用，那么存储在它附近的数据大概率也会很快被使用。`InnoDB的数据页`和`操作系统的页缓存`则是这种思路的体现。

## InnoDB行格式

mysql是以记录(一行数据)为单位向数据表中插入数据的，这些记录在磁盘上的存放方式称为`行格式`。mysql支持4种不同类型的行格式：`Compact`、`Redundant`（比较老，本文就不具体介绍了）、`Dynamic`、`Compressed`。
我们可以在创建或修改表的语句中指定行格式：

```sql
CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
ALTER TABLE 表名 ROW_FORMAT=行格式名称
```

比如，我们要创建一个行格式为`Compact`，字符集为`ascii`的数据表`record_format_demo`，sql如下：

```sql
mysql> CREATE TABLE record_format_demo (
    ->     c1 VARCHAR(10),
    ->     c2 VARCHAR(10) NOT NULL,
    ->     c3 CHAR(10),
    ->     c4 VARCHAR(10)
    -> ) CHARSET=ascii ROW_FORMAT=COMPACT;
Query OK, 0 rows affected (0.03 sec)
```

假设我们向`record_format_demo`表中插入了2行数据：

```shell
mysql> SELECT * FROM record_format_demo;
+------+-----+------+------+
| c1   | c2  | c3   | c4   |
+------+-----+------+------+
| aaaa | bbb | cc   | d    |
| eeee | fff | NULL | NULL |
+------+-----+------+------+
2 rows in set (0.00 sec)
```

### COMPACT行格式

![COMPACT行格式](https://user-gold-cdn.xitu.io/2019/3/12/169710e8fafc21aa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从上图可以看出，一条完整的记录包含`记录的额外信息`和`记录的真实数据`两大部分。

#### 记录的额外信息

记录的额外信息主要包含3类：`变长字段列表`、`NULL值列表`和`记录头信息`。

##### 变长字段列表

mysql中支持一些变长数据类型（比如`VARCHAR(M)`、`TEXT`等），它们存储数据占用的存储空间不是固定的，而是会随着存储内容的变化而变化。为了准确描述这种数据，这种变长字段占用的存储空间要同时包含：

1. 真正的数据内容
2. 占用的字节数

在Compact行格式中，把**所有变长字段的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表，各变长字段数据占用的字节数按照列的顺序`逆序`存放**。

我们以`record_format_demo`第一行数据为例。由于`c1`、`c2`和`c4`都是变成数据类型(`VARCHAR(10)`),因此要将这3列值得长度保存在记录的开头处。
![变长字段列表](https://user-gold-cdn.xitu.io/2019/3/12/169710e8fb363bb4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

另外需要注意的一点是，**变长字段长度列表中只存储值为 非NULL 的列内容占用的长度，值为 NULL 的列的长度是不储存的**。也就是说对于第二条记录来说，因为c4列的值为NULL，所以第二条记录的变长字段长度列表只需要存储c1和c2列的长度即可。
![变长字段列表](https://user-gold-cdn.xitu.io/2019/3/12/169710e8fe4ee6b0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### NULL值列表

对于可为NULL的列，为了节约存储空间，mysql不会将`NULL`值保存在`记录的真实数据`部分。而是会将其保存在`记录的额外信息`里面的`NULL值列表`中。

具体的做法是先统计表中允许存储`NULL`值的列，然后将每个允许存储`NULL`值的列对应一个二进制位（1：值为`NULL`，0：值不为`NULL`）用来表示是否存储`NULL`值，并按照逆序排列。MySQL规定**NULL值列表必须用整数个字节的位表示**，如果使用的二进制位个数不是整数个字节，则在字节的高位补0。
对应`record_format_demo`表中，`c1`、`c3`、`c4`都是允许存储NULL值的。前两条记录在填充了`NULL`值列表后的示意图就是这样：

![NULL值列表](https://user-gold-cdn.xitu.io/2019/3/12/169710e95903144f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 记录头信息

记录头信息是由固定的5个字节(40位)组成, 不同的位代表不同的含义：
![记录头信息](https://user-gold-cdn.xitu.io/2019/3/12/169710e97718ef01?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
暂时不详细展开。

### 记录的真实数据

记录的真实数据除了包含各列具体的数据外，还会自动添加一些隐藏列数据。

|列名|是否必须|占用空间|描述|
|-------|-------|-------|-------|
|row_id|否|6字节|行ID，唯一标识一条记录|
|transaction_id|是|6字节|事务ID|
|roll_pointer|是|7字节|回滚指针|

> 实际上这几个列的真正名称其实是：DB_ROW_ID、DB_TRX_ID、DB_ROLL_PTR，为了美观才写成了row_id、transaction_id和roll_pointer。

只有当数据库没有定义`主键`或者`唯一键`时，隐藏列`row_id`才会存在，并且将其作为数据表`主键`。
因为表`record_format_demo`并没有定义主键，所以MySQL服务器会为每条记录增加上述的3个列。现在看一下加上`记录的真实数据`的两个记录的数据结构：
![记录的真实数据](https://user-gold-cdn.xitu.io/2019/3/12/169710e973b70372?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### CHAR(M)列的存储格式

对于 CHAR(M) 类型的列来说，当列采用的是定长字符集时，该列占用的字节数不会被加到变长字段长度列表，而**如果采用变长字符集时，该列占用的字节数也会被加到变长字段长度列表**。
另外有一点还需要注意，变长字符集的`CHAR(M)`类型的列要求至少占用`M`个字节，而`VARCHAR(M)`却没有这个要求。比方说对于使用`utf8`字符集的`CHAR(10)`的列来说，该列存储的数据字节长度的范围是`10～30`个字节，即使我们向该列中存储一个空字符串也会占用`10`个字节。

### 行溢出数据

#### VARCHAR(M)最多能存储的数据

MySQL对一条记录占用的最大存储空间是有限制的，除了`BLOB`或者`TEXT`类型的列之外，**其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过65535个字节**。可以不严谨的认为，**mysql一行记录占用的存储空间不能超过65535个字节**。这个65535个字节除了列本身的数据之外，还包括一些其他的数据（storage overhead），比如说我们为了存储一个VARCHAR(M)类型的列，其实需要占用3部分存储空间：

1. 真实数据
2. 真实数据占用字节的长度
3. NULL值标识，如果该列有NOT NULL属性则可以没有这部分存储空间

假设`varchar_size_demo`只有一个`VARCHAR`类型的字段，那么该字段最大占用的65532个字节。因为真实数据的长度可能占用2个字节，`NULL值标识`需要占用1个字节。如果该`VARCHAR`类型的列没有`NOT NULL`属性，那最多只能存储`65532`个字节的数据。如果该列是`ascii`字符集，对应的最大字符数最大为`65532`；如果是`utf8`字符集，则对应的最大字符数为`21844`。

#### 记录中的数据太多产生的溢出

我们以ascii字符集下的`varchar_size_demo`表为例，插入一条记录：

```sql
mysql> CREATE TABLE varchar_size_demo(
    ->       c VARCHAR(65532)
    -> ) CHARSET=ascii ROW_FORMAT=Compact;
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO varchar_size_demo(c) VALUES(REPEAT('a', 65532));
Query OK, 1 row affected (0.00 sec)
```

mysql中磁盘与内存交互的基本单位是页，一般为16KB，16384个字节，而一行记录最大可以占用`65535`个字节，这就造成了**一页存不下一行数据的情况**。在Compact和Redundant行格式中，对于占用存储空间非常大的列，在记录的真实数据处只会存储该列的一部分数据，把剩余的数据分散存储在几个其他的页中，然后记录的真实数据处用20个字节存储指向这些页的地址，从而可以找到剩余数据所在的页，如图所示：
![行溢出数据](https://user-gold-cdn.xitu.io/2019/3/12/169710e9aab47ea5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
这种**在本记录的真实数据处只会存储该列的前768个字节的数据和一个指向其他页的地址，然后把剩下的数据存放到其他页中的情况就叫做`行溢出`，存储超出768字节的那些页面也被称为`溢出页`**。
![行溢出数据](https://user-gold-cdn.xitu.io/2019/3/12/169710e9a5d5637a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 行溢出的临界点

**MySQL中规定一个页中至少存放两行记录**。以上边的`varchar_size_demo`表为例，它只有一个列`c`，我们往这个表中插入两条记录，每条记录最少插入多少字节的数据才会行溢出的现象呢？这得分析一下页中的空间都是如何利用的。

1. 每个页除了存放我们的记录以外，也需要存储一些额外的信息，大概132个字节。
2. 每个记录需要的额外信息是27字节。

假设一个列中存储的数据字节数为n，如要要保证该列不发生溢出，则需要满足：

```c
132 + 2×(27 + n) < 16384
```

结果是`n < 8099`。**也就是说如果一个列中存储的数据小于8099个字节，那么该列就不会成为溢出列**。如果表中有多个列，那么这个值更小。

### Dynamic和Compressed行格式

mysql中默认的行格式就是`Dynamic`。`Dynamic`和`Compressed`行格式和`Compact`行格式很像，只是在处理`行溢出`数据上有差异。`Dynamic`和`Compressed`行格式不会在`记录的真实数据`出存放前768个字节，而是将所有字节都存储在其它页面中。`Compressed`行格式会采用压缩算法对页面进行压缩，以节省空间。
![Dynamic](https://user-gold-cdn.xitu.io/2019/3/12/169710e9b2c2b71e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## InnoDB数据页结构

我们已经知道页是InnoDB管理存储空间的基本单位，一个页的大小一般是16KB。InnoDB为了不同的目的设计了许多不同类型的页，我们这里主要关注`存储数据记录`的页，官方称为`索引页`。由于还没介绍索引，暂且我们先称为`数据页`吧。

### 数据页结构的快速浏览

数据页在结构上可以划分为多个部分，不同的部分有不同的功能，如下图所示：
![数据页结构](https://user-gold-cdn.xitu.io/2019/12/17/16f13ee1e2dfac7c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

一个InnoDB数据页被划分为了7个部分，下面大概描述一下这7个部分内容。

|名称|中文名|占用空间大小|简单描述|
|-----|-----|-----|-----|
|File Header|文件头部|38字节|页的一些通用信息|
|Page Header|页面头部|56字节|数据页专有的一些信息|
|Infimum + Supremum|最小记录和最大记录|26字节|两个虚拟的行记录|
|User Records|用户记录|不确定|实际存储的行记录内容|
|Free Space|空闲空间|不确定|页中尚未使用的空间|
|Page Directory|页面目录|不确定|页中的某些记录的相对位置|
|File Trailer|文件尾部|8字节|校验页是否完整|

### 记录在页中的存储

用户自己的存储的数据会按照对应的`行格式`存在`User Records`中。实际上，新生成的页面是没有`User Records`的，只有当我们第一次插入数据时，才会从`Free Space`划一个记录大小的空间给`User Records`。当`Free Space`用完之后，就意味着当前的数据页也使用完了。
![记录在页中的存储](https://user-gold-cdn.xitu.io/2019/5/8/16a95c0fe86555ed?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
为了能够将`User Records`讲清楚，我们先得理解前面提到的`记录头信息`。

#### 理解记录头信息

先简单介绍一下记录头信息各属性描述：

|名称|大小（单位：bit）|描述|
|-------|-------|-------|
|预留位1|1|没有使用|
|预留位2|1|没有使用|
|delete_mask|1|标记该记录是否被删除|
|min_rec_mask|1|B+树的每层非叶子节点中的最小记录都会添加该标记|
|n_owned|4|表示当前记录拥有的记录数|
|heap_no|13|表示当前记录在记录堆的位置信息|
|record_type|3|表示当前记录的类型，0表示普通记录，1表示B+树非叶子节点记录，2表示最小记录，3表示最大记录|
|next_record|16|表示下一条记录的相对位置|

接下来以`page_demo`表为例，并插入一些数据,详细介绍记录头信息。

```sql
mysql> CREATE TABLE page_demo(
    ->     c1 INT,
    ->     c2 INT,
    ->     c3 VARCHAR(10000),
    ->     PRIMARY KEY (c1)
    -> ) CHARSET=ascii ROW_FORMAT=Compact;
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO page_demo VALUES(1, 100, 'aaaa'), (2, 200, 'bbbb'), (3, 300, 'cccc'), (4, 400, 'dddd');
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

这4条记录在InnoDB中的行格式如下（只展示记录头和真实数据），列中数据均用十进制表示：
![记录头](https://user-gold-cdn.xitu.io/2019/5/8/16a95c0ff83f9870?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
我们对照着这个图来重点介绍几个属性的详细信息：

- `delete_mask`：标记着当前记录是否被删除，0表示未删除，1表示删除。未删除的记录不会立即从磁盘上移除，而是先打上删除标记，所有被删除的记录会组成一个`垃圾链表`。之后新插入的记录可能会重用`垃圾链表`占用的空间，因此垃圾链表占用的存储空间也被成为`可重用空间`。
- `heap_no`：表示当前记录在本页中的位置，比如上边4条记录在本页中的位置分别是`2、3、4、5`。实际上，InnoDB会自动为每页加上两条虚拟记录，一条是`最小记录`，另一条是`最大记录`。这两条记录的构造十分简单，都是由`5字节大小的记录头信息`和`8字节大小的固定部分`(其实内容就是infimum或者supremum)组成的。这两条记录被单独放在`Infimum + Supremum`的部分。
  ![heap_no](https://user-gold-cdn.xitu.io/2019/5/8/16a95c10773d8cee?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
  从图中我们可以看出来，最小记录和最大记录的`heap_no`值分别是0和1，也就是说它们的位置最靠前。
- `next_record`：表示从**当前记录的真实数据到下一条记录的真实数据的地址偏移量**。可以简单理解为是一个单向链表，最小记录的下一个是第一条记录，最后一条记录的下一个是最大记录。为了更加形象的展示，我们可以用箭头来替代一下next_record中的地址偏移量：
  ![next_record](https://user-gold-cdn.xitu.io/2019/5/8/16a95c1084c440b4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
  从图中也能看出来，**用户记录实际上按照主键大小正序排序行成一个单向链表**。如果从中删除掉一条记录，这个链表也是会跟着变化的，比如我们把第2条记录删掉：
  ![next_record](https://user-gold-cdn.xitu.io/2019/5/8/16a95c108ee1da43?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
  - 第2条记录并没有从存储空间中移除，而是把该条记录的`delete_mask`值设置为1。
  - 第2条记录的`next_record`值变为了0，意味着该记录没有下一条记录了。
  - 第1条记录的`next_record`指向了第3条记录。

### Page Directory（页目录）

我们已经知道，记录在页中按照主键大小正序串联成了一个单链表。如果我们要根据主键查找具体的某条记录应该怎么办，简单的方式是根据链表进行遍历。但是在数据量比较大的情况下，这种方式显然效率太差了。因此mysql使用了`Page Directory（页目录）`来解决这个问题。`Page Directory（页目录）`大致的原理如下：

1. 将所有正常的记录（包括最大和最小记录，不包括标记为已删除的记录）划分为几个组。怎么划分先不关注。
2. 每个组的最后一条记录（也就是组内最大的那条记录）的头信息中的n_owned属性表示该组内共有几条记录。
3. 将每个组的最后一条记录的地址偏移量单独提取出来按顺序存储到靠近页尾部的地方，这个地方就是所谓的`Page Directory`。

**mysql规定对于最小记录所在的分组只能有 1 条记录，最大记录所在的分组拥有的记录条数只能在 1-8 条之间，剩下的分组中记录的条数范围只能在是 4-8 条之间**。
比方说现在的`page_demo`表中正常的记录共有18条，InnoDB会把它们分成5组，第一组中只有一个最小记录，如下所示：
![Page Directory](https://user-gold-cdn.xitu.io/2019/5/8/16a95c10e3449897?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

通过`Page Directory`在一个数据页中查找指定主键值的记录的过程分为两步：

1. **通过二分法确定该记录所在的槽，并找到该槽所在分组中主键值最小的那条记录。**
2. **通过记录的`next_record`属性遍历该槽所在的组中的各个记录。**

> 对于链表的查询性能优化，思想上基本上都是通过`二分法`实现的。上面介绍的`Page Directory`，`跳跃表`和`查找树`都是如此。

### Page Header（页面头部）

`Page Header`专门用来存储数据页相关的各种状态信息，比如本页中已经存储了多少条记录，第一条记录的地址是什么，页目录中存储了多少个槽等等。固定占用56个字节，各部分字节属性含义如下：

|名称|占用空间大小|描述|
|-----|-----|-----|
|PAGE_N_DIR_SLOTS|2字节|在页目录中的槽数量|
|PAGE_HEAP_TOP|2字节|还未使用的空间最小地址，也就是说从该地址之后就是`Free Space`|
|PAGE_N_HEAP|2字节|本页中的记录的数量（包括最小和最大记录以及标记为删除的记录）|
|PAGE_FREE|2字节|第一个已经标记为删除的记录地址（各个已删除的记录通过next_record也会组成一个单链表，这个单链表中的记录可以被重新利用）|
|PAGE_GARBAGE|2字节|已删除记录占用的字节数|
|PAGE_LAST_INSERT|2字节|最后插入记录的位置|
|PAGE_DIRECTION|2字节|最后一条记录插入的方向|
|PAGE_N_DIRECTION|2字节|一个方向连续插入的记录数量，如果最后一条记录的插入方向改变了的话，这个状态的值会被清零重新统计。|
|PAGE_N_RECS|2字节|该页中记录的数量（不包括最小和最大记录以及被标记为删除的记录）|
|PAGE_MAX_TRX_ID|8字节|修改当前页的最大事务ID，该值仅在二级索引中定义|
|PAGE_LEVEL|2字节|当前页在B+树中所处的层级|
|PAGE_INDEX_ID|8字节|索引ID，表示当前页属于哪个索引|
|PAGE_BTR_SEG_LEAF|10字节|B+树叶子段的头部信息，仅在B+树的Root页定义|
|PAGE_BTR_SEG_TOP|10字节|B+树非叶子段的头部信息，仅在B+树的Root页定义|

这里只是罗列出来，暂时不需要全部理解。

### File Header（文件头部）

`File Header`是用来描述各种页都适用的一些通用信息的，由以下内容组成：

|名称|占用空间大小|描述|
|-----|-----|-----|
|FIL_PAGE_SPACE_OR_CHKSUM|4字节|页的校验和（checksum值）|
|FIL_PAGE_OFFSET|4字节|页号|
|FIL_PAGE_PREV|4字节|上一个页的页号|
|FIL_PAGE_NEXT|4字节|下一个页的页号|
|FIL_PAGE_LSN|8字节|页面被最后修改时对应的日志序列位置（英文名是：Log Sequence Number）|
|FIL_PAGE_TYPE|2字节|该页的类型|
|FIL_PAGE_FILE_FLUSH_LSN|8字节|仅在系统表空间的一个页中定义，代表文件至少被刷新到了对应的LSN值|
|FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID|4字节|页属于哪个表空间|

这里只是罗列出来，暂时不需要全部理解。我们重点关注一下几个属性：

1. `FIL_PAGE_SPACE_OR_CHKSUM`
  当前页面的校验和（checksum）。对于一个很长的字节串来说，我们可以通过某种算法来计算一个比较短的值来代表这个很长的字节串，这个比较短的值就称为`校验和`。通过`校验和`可以大幅度提升字符串等值比较的效率。
2. `FIL_PAGE_OFFSET`
   每一个页都有一个唯一的页号，`InnoDB`通过页号来可以定位一个页。
3. `FIL_PAGE_TYPE`
   代表当前页的类型，我们前边说过，InnoDB为了不同的目的而把页分为不同的类型。
   |类型名称|十六进制|描述|
   |-----|-----|-----|
   |FIL_PAGE_TYPE_ALLOCATED|0x0000|最新分配，还没使用|
   |FIL_PAGE_TYPE_ALLOCATED|0x0000|最新分配，还没使用|
   |FIL_PAGE_UNDO_LOG|0x0002|Undo日志页|
   |FIL_PAGE_INODE|0x0003|段信息节点|
   |FIL_PAGE_IBUF_FREE_LIST|0x0004|Insert Buffer空闲列表|
   |FIL_PAGE_IBUF_BITMAP|0x0005|Insert Buffer位图|
   |FIL_PAGE_TYPE_SYS|0x0006|系统页|
   |FIL_PAGE_TYPE_TRX_SYS|0x0007|事务系统数据|
   |FIL_PAGE_TYPE_FSP_HDR|0x0008|表空间头部信息|
   |FIL_PAGE_TYPE_XDES|0x0009|扩展描述页|
   |FIL_PAGE_TYPE_BLOB|0x000A|溢出页|
   |FIL_PAGE_INDEX|0x45BF|索引页，也就是我们所说的数据页|
4. `FIL_PAGE_PREV`和F`IL_PAGE_NEXT`
   表示本页的上一个和下一个页的页号，各个页通过`FIL_PAGE_PREV`和F`IL_PAGE_NEXT`形成双向链表。
   ![FIL_PAGE_PREV](https://user-gold-cdn.xitu.io/2019/5/8/16a95c10eb9d61ce?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### File Trailer

mysql中内存和磁盘的基本交互单位是页。如果内存中页被修改了，那么某个时刻一定会将内存页同步到磁盘中。如果在同步的过程中，系统出现问题，就可能导致磁盘中的页数据没能完全同步，也就是发生了`脏页`的情况。为了避免发生这种问题，mysql在每个页的尾部加上了`File Trailer`来校验页的完整性。`File Trailer`由8个字节组成：

1. 前4个字节代表页的校验和
   这个部分是和File Header中的校验和相对应的。简单理解，就是`File Header`和`File Trailer`都有校验和，如果两者一致则表示数据页是完整的。否则，则表示数据页是`脏页`。
2. 后4个字节代表页面被最后修改时对应的日志序列位置（LSN）
   这个部分也是为了校验页的完整性的，暂不详细了解。

> 本文主要内容是根据掘金小册《从根儿上理解 MySQL》整理而来。如想详细了解，建议购买掘金小册阅读。

