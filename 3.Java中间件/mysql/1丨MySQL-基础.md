# 1.【MySQL概述】

## 1.1.体系结构

如下图，按照从上到下，从左到右的顺序，MySQL的结构大体上分为：

- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件
- 优化器组件
- 缓冲组件
- 插件式存储引擎
- 物理文件

![](./images/mysql体系结构.png)

其中，比较突出的就是MySQL的可拔插存储引擎，是底层物理存储的实现。需要注意的是，**MySQL的存储引擎是基于表的，而不是基于数据库的**。除了常用的`InnoDB`外，还有这些常见的MySQL存储引擎：

- MyISAM：不支持事务，采用表锁设计，支持全文索引
- NDB：集群存储引擎，数据放置于内存中，可提供更高的可用性
- Memory：表中数据存放在内存中，采用哈希索引，适用于临时数据的临时表
- Archive：只支持insert和select操作，使用存储归档数据，提供高速的插入和压缩功能
- Federated：不存放数据，指向一台远程MySQL数据库服务器上的表
- Maria：支持缓存数据和索引文件，行锁设计，提供MVCC更，支持事务

## 1.2.常用日志

**mysql server**

- **binlog** - 归档日志，也称为二进制日志。主要存储行记录的变化，binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。binlog有3种数据格式：statement、row、mixed，其中statement会原封不动地记录执行的SQL语句，row会记录SQL执行前后的实际数据变化，而mixed是statement和row的结合。因为statement记录的是SQL，主从数据库执行SQL时有可能会造成数据不一样，而row格式会记录大量的实际数据，所以mysql设计了mixed格式，如果它觉得此次SQL不会造成主备不一致它就使用statement格式，反之使用row格式；
- **error log** - 错误日志。对mysql的启动、运行、关闭过程进行了记录，不仅记录了所有的错误信息，也记录了一些警告信息或正确的信息，可以通过`show variables like'log_error'`定位错误日志文件位置；
- **slow log** - 慢查询日志。通过参数`long_query_time`来设定一个阈值，默认为10，代表10秒。表示将运行时间超过该值（大于）的所有SQL语句都记录到慢查询日志中；

**InnoDB**

- **redo log** - 重做日志，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为**crash-safe**。重做日志是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB，那么总共就可以记录4GB的操作。从头开始写，写到末尾就又回到开头循环写

## 1.3.SQL类别

SQL 语句，全称为Structure Query Language（结构化查询语言）主要分为以下 3 个类别：

- **DDL（Data Definition Language）语句：**数据定义语言，这些语句定义了不同的数据段、数据库、表、列、索引等数据库对象的定义。常用的语句关键字主要包括 create、drop、alter等

- **DML（Data Manipulation Language）语句：**数据操纵语句，用于添加、删除、更新和查询数据库记录，并检查数据完整性，常用的语句关键字主要包括 insert、delete、udpate 和select 等(增添改查）

- **DCL（Data Control Language）语句：**数据控制语句，用于控制不同数据段直接的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。主要的语句关键字包括 grant、revoke 

# 2.【InnoDB存储引擎】

## 2.1.体系结构

InnoDB存储引擎是MySQL在5.5.8版本之后默认使用的，特点是：支持事务，行锁设计，支持外键。InnoDB通过使用多版本并发控制（MVCC）获得高并发性，并且实现了SQL标准的4种隔离级别，默认为REPEATABLE级别。InnoDB的架构，包括两大部分，内存结构（In-Memory Structures）和磁盘上的结构（On-Disk Structures）

![](./images/InnoDB体系结构.jpeg)

如果单看 InnoDB内存数据对象，就如下图所示：

![](./images/InnoDB体系结构.jfif)

### 2.1.1.后台线程

InnoDB有多个后台线程，负责处理不同的任务，大致上分为：

- **Master Thread**：将缓冲池的数据异步刷新到磁盘，包括脏页的刷新、合并插入缓冲、UNDO页的回收；
- **IO Thread**：InnoDB存储引擎使用AIO处理IO请求，IO Thread负责这些IO请求的回调处理
- **Purge Thread**：事务提交后，其undolog便失去作用，Purge Thread负责回收这些undo页；减轻Master Thread的工作
- **Page Cleaner Thread**：负责将脏页的刷新操作放入到单独的线程中，减轻Master Thread的工作

### 2.1.2.缓冲池

由于磁盘速度的影响，通常系统设计会用内存作为CPU和磁盘之间的纽带，也就是缓冲区，Buffer Pool就 是Innodb 内存中的的一块比较大的区域，用来缓存表和索引数据。Buffer Pool可以加速SQL查询的效率，大小是由参数`innodb_buffer_pool_size`确定，一般建议设置成可用物理内存的60%~80%。如果是写操作，首先修改缓冲池的页，再通过Checkpoint机制刷回磁盘；如果是读操作，优先读取缓冲池中的页，若页不存在再去读取磁盘，然后将页保存到缓冲池中。正是由于每次都是基于内存来执行SQL，所以mysql的吞吐量很高！

Buffer pool是按照页（Page）来分配的，受到`innodb_page_size`控制，默认大小为16KB。Buffer pool存在3种类型的数据页，每一种页存在于它对应的链表：

- **free page**：空闲页，从未使用过的页，位于Free链表。当需要从缓冲池中取页时，先到Free List查找是否有可用的空闲页，若有则从Free List中删除，放入到LRU List中；否则从LRU List末尾删除页，将该内存空间分配给新读取的页，这一操作称为`page made young`；
- **clean page**：干净页，已被使用过的页，但是页数据未被修改，即页的数据和磁盘是一致的，处于LRU链表。随着程序的运行，缓冲池中的页会越来越多，因此InnoDB基于传统LRU算法，加入midpoint位置来管理缓冲池的页，这个算法称为`midpoint insertion strategy`，作用是：在原有LRU基础上，最新访问的页并不是直接放入到列表首部，而是放到midpoint位置
- **dirty page**：脏页，已被使用过的页，并且页数据已被修改，当脏页上的数据写入磁盘后，脏页又会变成干净页。脏页同时存在于LRU链表和Flush链表。LRU 链表中的页被修改后，该页就会变为脏页，意味着缓冲池和磁盘的页数据产生了不一致。InnoDB的方案是通过Checkpoint机制将脏页刷回到磁盘，这些脏页是存放在Flush List中，不过要注意的是，脏页既存在于Flush List，它还仍存放在LRU List中

InnoDB内存管理用的是最近最少使用 (Least Recently Used, LRU)算法，这个算法的核心就是淘汰最久未使用的数据。但是InnoDB的LRU算法跟基本的LRU算法不一样，原因之一就是：有些SQL查询的时候使用其它非活跃数据的辅助（比如索引和数据的扫描），如果直接将这些非活跃数据放到LRU列表首部，那么很有可能把真正活跃的热点数据挤出去！！那么业务查询的时候Buffer Pool的**内存命中率**将会急剧下降，磁盘压力就增加，SQL响应的速度就会变慢。在实现上，它按照5:3的比例把整个LRU链表分成了young区域和old区域。图中LRU_old指向的就是old区域的第一个位置，是整个链表的5/8处。也就是说，靠近链表头部的5/8是young区域，靠近链表尾部的3/8是old区域，中间这个LRU_old称为midpoint

![](./images/mysql-改进的LRU算法.png)

改进后的LRU算法执行流程：

- 要访问数据页P3，由于P3在young区域，这种情况就和普通LRU算法一样，将其移到链表头部，变为状态2
- 如果后续访问到一个不存在当前链表的数据页，就会淘汰old区域中tail指针指向的数据页pm，然后再LRU_old指向的位置插入新的数据页px，如状态3所示
- 如果处于old区域的数据页，每次被访问的时候，做如下判断：
  - 若这个数据页在LRU链表中存在的时间超过1秒，将其移到链表头部
  - 若这个数据页在LRU链表中存在的时间短于1秒，位置保持不变。

通过命令可以查看LRU算法的两个核心配置，`innodb_old_blocks_pct`参数将LRU链表分为两部分，一部分是热点数据，一部分是冷数据，默值为37%表，表示old区域占比。`innodb_old_blocks_time`参数来控制成为热数据的所需时间，默认是1000ms，也就是1s，数据在1s内没有被刷走，就就会被移到young区。

```sql
mysql> show variables like '%old_blocks%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_pct  | 37    |
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
2 rows in set (0.01 sec)
```

### 2.1.3.重做日志缓冲

重做日志缓冲，redo log buffer，InnoDB首先将重做日志信息先放入到这个缓冲，然后按照一定频率将其刷新到重做日志文件。一般情况下，每一秒钟会将重做日志缓冲刷新到磁盘中，用户只需要保证每秒产生的事务量在这个缓冲大小之内即可

```sql
-- 默认是8MB
SHOW VARIABLES LIKE 'innodb_log_buffer_size'
```

在下列三种情况下，会将重做日志缓冲中的内容刷新到磁盘中：

- Master Thread每秒将重做日志缓冲刷新到磁盘中的重做日志文件
- 每个事务提交时会将重做日志缓冲刷新重做日志文件
- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件

## 2.2.存储结构

所有数据都被逻辑存放到表空间（Tablespace）中，而一个表空间又可以分为段（Segment）、区（extent）、页（page），如下图所示：

![](./images/InnoDB存储结构.png)

### 2.2.1.表空间

默认配置，InnoDB存储引擎会使用一个共享表空间ibdata1，所有的表数据都会存放在这里。但如果用户启用了参数innodb_file_per_table，则每张表的数据都可以存放各自的表空间中。不过，每张表自己的表空间只存放数据、索引、插入缓冲Bitmap页。

### 2.2.2.段

表空间tablespace是由各个段segment组成，常见的段有：数据段、索引段、回滚段等。数据段为B+树的叶子节点，索引段位B+树的非叶子节点

### 2.2.3.区

每个段segment存在多个区extent，区是由连续的页组成的，每个区的大小固定为1MB，因为页的大小为16KB，所以一个区中一共会有64个连续的页。

### 2.2.4.页

页Page是InnoDB磁盘管理的最小单位，默认每个页的大小为16KB，InnoDB的数据是按数据页为单位来读写的。也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存。在InnoDB中，每个数据页的大小默认是16KB。常见的页类型有：

- 数据页，B-tree Node
- undo页，undo log page
- 系统页，system page
- 事务数据页，Transaction system page
- 插入缓冲位图页，Insert Buffer Bitmap
- 插入缓冲空闲列表页，Insert Buffer Free List
- 未压缩的二进制大对象页，Uncompressed BLOB Page
- 压缩的二进制大对象页，compressed BLOB Page

## 2.3.关键特性

### 2.3.1.写缓冲(插入缓冲)

在mysql 5.5版本以前，支持insert操作，所以这个缓冲也称为`插入缓冲`，后面版本支持更多的操作类型缓存，改称为`写缓冲(change buffer)`。在InnoDB中，主键是行唯一的标识符，行记录的插入顺序是按照主键递增的顺序进行插入的，这种情况不需要读取其它页就可以完成insert操作；但是如果主键是UUID这种随机数，或者表中存在二级索引，InnoDB对这种数据的插入，必须离散地访问非聚集索引页，由于随机读取而导致插入性能下降。

基于这一考虑，InnoDB设计了change buffer，当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页。然后，再以一定的频率将change buffer中的操作应用到原数据页，这一过程称为`merge`，通常能将多个插入`merge`到一个操作中（因为在一个索引页中），这就大大提高了对于非聚集索引插入的性能！这个设计思路和HBase中的LSM树有相似之处，都是通过先在内存中修改，到达一定量后，再和磁盘中的数据合并，目的都是为了提高写性能，具体可参考《[HBase LSM树](https://zhuanlan.zhihu.com/p/135371171)》

- **InnoDB在同时满足以下两个条件就会使用change buffer**
  - 索引是辅助索引（secondary index）
  - 索引不是唯一（unique）的，唯一索引需要获取该页所有数据来加载判断是否存在

- **InnoDB有3种情况会触发merge操作**

  - 访问这个数据页会触发merge；

  - 系统有后台线程会定期merge；

  - 数据库正常关闭（shutdown）的过程中，也会执行merge操作；

- **change buffer执行流程**

  假设要向一张表中插入一条新纪录(id=4, value=400)，使用channge buffer的流程如下：

  第一种情况是，**这个记录要更新的目标页在内存中**。这时，InnoDB的处理流程如下：（区别不大）

  - 对于唯一索引来说，找到3和5之间的位置，判断到没有冲突，插入这个值，语句执行结束；
  - 对于普通索引来说，找到3和5之间的位置，插入这个值，语句执行结束。

  第二种情况是，**这个记录要更新的目标页不在内存中**。这时，InnoDB的处理流程如下：（区别大了）

  - 对于唯一索引来说，需要将数据页读入内存（走磁盘），判断到没有冲突，插入这个值，语句执行结束；
  - 对于普通索引来说，则是将更新记录在change buffer，语句执行就结束了。

- **merge执行流程**

  - 从磁盘读入数据页到内存（此时的数据页是旧数据）；
  - 从change buffer里找出这个数据页的change buffer记录（可能会有多个），依次执行这些记录，就会得到最新的数据页；

  - 写redo，此次写入包含了数据的变更和change buffer的变更；

  执行完前三步，merge流程结束。但是此时数据页和change buffer对应的的磁盘位置还没有修改，属于脏页，Innodb会通过两次写来将脏页刷入到磁盘中，当然这就是属于另一个过程；

- **实际场景**

  - 对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用change buffer。

  - 对于写多读少的业务（账单类、日志类业务）来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好；但是，如果一个业务的更新模式是写入之后马上会做查询，即使将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程，反而增加了change buffer的维护代价，从而起到了副作用

### 2.3.2.两次写

doublewrite，两次写，是InnoDB保证数据页落地到磁盘中的解决方案。假设，InnoDB存储引擎正在将数据页从缓冲池中写入到磁盘，刚写入4KB的时候就发生宕机，这种情况被称为部分写失效（partial page write）这是没办法通过重做日志（redo）进行恢复的，因为redo log要先对磁盘上的页进行读取，而现在是这个页已经损坏了。为了解决这个问题，InnoDB存储引擎开发了doublewrite功能，它的结构为：

- 处在内存中的doublewrite buffer，大小2MB
- 处于物理磁盘上共享表空间中连续的128个页，即2个extent，大小2MB

![](./images/InnoDB-doublewrite架构.png)

解决方案是这样：

- 再对缓冲池的数据页进行刷新的时候，并不会直接写到磁盘上，而是先将数据页复制到doublewrite buffer，再通过doublewrite buffer分两次，每次1MB顺序地写入共享表空间的物理磁盘上。最后同步磁盘，避免缓冲写带来的问题
- 如果将数据页刷新到磁盘的过程中发生了崩溃，InnoDB先从共享表空间中的doublewrite找到改页的一个副本，将其复制到表空间文件，再应用重做日志

### 2.3.3.自适应哈希索引

InnoDB会监控表上各索引页的查询，根据访问的频率和模式来自动地为某些热点页建立哈希索引，这个就称为自适应哈希索引，Adaptive Hash Index，AHI。哈希索引只能用来搜索等值的查询，且要求查询的条件是一样的。

### 2.3.4.异步I/O

InnoDB采用异步IO（Asynchronous IO，AIO）的方式来处理磁盘操作，现版本InnoDB采用了内核级别的AIO，称为Native AIO，它需要底层操作系统提供支持。AIO除了可以快速处理多个IO请求外（Sync IO，同步IO必须在一个页扫描完以后再进行下一次扫描），还可以进行IO Merge，将多个IO合并为1个IO。

### 2.3.5.刷新邻接页

Flush Neighbor Page，刷新邻接页。当刷新一个脏页时，InnoDB存储引擎会检测该页所在区（extent）的所有页，如果是脏页，那么一起进行刷新

### 2.3.6.Checkpoint机制

Checkpoint机制用来解决缓冲池跟磁盘之间的数据一致性问题。InnoDB对数据页的操作，都是先操作缓冲池，这就会引发一个问题：若每次数据页发生变化，InnoDB就将新的数据页刷入到磁盘中，会导致开销很大；但是，数据页不及时刷入到磁盘中，若数据库实例宕机，那么内存中的数据页就丢失了。为了避免这一问题，大部分事务数据库系统普遍使用`Write Ahead Log`策略（即WAL）：当事务提交时，先写重做日志(redo log)，再修改缓冲池的数据页，这样即使数据库实例宕机，也可以通过重做日志来恢复数据，当然如果连写入日志都失败了，那么这个事务肯定就是属于执行失败的情况了。

当**redo log写满**了，系统就会停止所有更新操作，把checkpoint往前推进：位置从CP推进到CP’，就需要将两个点之间的日志（浅绿色部分）对应的所有脏页都刷新到磁盘上。之后，图中从write pos到CP’之间就是可以再写入的redo log的区域：

![](./images/checkpoint机制.jpg)

# 3.【MySQL扩展点】

## 3.1.客户端数据传输

mysql服务端查询得到的数据，是如何发给mysql客户端的？答案是：**边读边发**。当执行器从存储引擎中获取到数据后，它会执行如下的步骤：

1. 获取一行，写到net_buffer中，这块内存的大小是由参数net_buffer_length定义的，默认是16k;
2. 重复获取行，直到net_buffer写满，调用网络接口发出去;
3. 如果发送成功，就清空net_buffer，然后继续取下一行，并写入net_buffer;
4. 如果发送函数返回EAGAIN或WSAEWOULDBLOCK，就表示本地网络栈（socket send buffer）写满了，进入等待。直到网络栈重新可写，再继续发送。

![](./images/查询结果发送流程.jpg)

由于mysql是边读边发，所以如果客户端接收得慢，MySQL服务端由于查询结果发不出去，就会导致SQL的执行时间变长。在mysql服务通过执行`show processlist`，可以看到State状态有如下几种情况：

- Sending to client：表示正发送数据给客户端，如果一直处于这一状态就说明服务端的网络栈写满了，mysql服务端一直在等待客户端接收sql查询结果
- Sending data：当sql查询语句进入执行阶段时，state就会改为此状态，直到执行完成。

MySQL客户端发送请求后，接收服务端返回结果的方式有两种：

1. 一种是本地缓存，也就是在本地开一片内存，先把结果存起来。如果用mysql API开发，对应的就是`mysql_store_result` 方法。
2. 另一种是不缓存，读一个处理一个，如果用mysql API开发，对应的就是`mysql_use_result`方法。

MySQL客户端默认采用第一种方式，而如果加上–quick参数，就会使用第二种不缓存的方式。采用不缓存的方式时，如果本地处理得慢，就会导致服务端发送结果被阻塞，因此会让服务端变慢。**对于正常的线上业务来说，如果一个查询的返回结果不会很多的话，一般使用mysql_store_result这个接口，直接把查询结果保存到本地内存。**

## 3.2.临时表

在mysql中，临时表并不就是内存表，这两者是有区别的：

- **内存表**：是指用Memory引擎的表，建表语法为`create table ... engine=memory`。这种表的数据都存储在内存中，系统重启就会被清空，但是表结构还存在；

- **临时表**：可以使用各种引擎类型，建表语法为`create temporary table ...`。若使用innoDB引擎或MyISAM引擎的临时表，写数据的时候就会写到磁盘上；如果使用的Memory引擎，写数据的时候就只写到内存中。

临时表有如下几个特点：

1. 一个临时表只能被创建它的线程访问，对其它线程不可见；
2. 临时表可以与普通表同名，在磁盘存储上，临时表会加上额外标识
3. 一个线程内有同名的临时表和普通表时，`show create`语句，以及增删改查语句访问的是临时表；
4. `show tables`命名不会显示临时表；
5. 当线程断开（即session结束），会自动删除它创建的临时表；

每个线程都维护了自己的临时表链表。这样每次session内操作表的时候，先遍历链表，检查是否有这个名字的临时表，如果有就优先操作临时表，如果没有再操作普通表；在session结束的时候，对链表里的每个临时表，执行 “DROP TEMPORARY TABLE +表名”操作。只有当binlog_format=statment/mixed 的时候，binlog才会记录临时表的操作！！

## 3.3.内存表

内存表一般使用 Memory 引擎来构建，sql语法为`create table ... engine=Memory;`，而且InnoDB和Memory引擎的数据组织方式是不同的：

1. InnoDB引擎把数据放在主键索引上，其它索引上保存的是主键id。这种方式，称之为**索引组织表**（Index Organizied Table）
2. Memory引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，这种称之为**堆组织表**（Heap Organizied Table）

![](./images/Memory引擎-Hash索引.jpg)

内存表的数据以数组的方式单独存放，而在索引里，存的是每个数据的具体位置。很明显，主键id走的是hash索引，并且索引上的key并不是有序（InnoDB引擎用的B+树索引是有序的），但实际上，内存表也是可以支持B+树索引的，语法就是手动指定用的索引方式：

```sql
alter table t1 add index a_btree_index using btree (id);
```

![](./images/Memory引擎-Btree索引.jpg)

内存表读写速度的原因有两个：其一，走内存比走磁盘块；其二，内存表直接hash索引，比B-tree索引快。不过生产环境上没人会用内存表来存储数据（除非一些作为中间过渡用的情况），主要是因为：

- 内存表不支持行锁，只有表锁。所以导致它的并发度并不高；
- 数据库重启时，所有内存表都会被清空。一旦线上环境搭建的是mysql高可用架构，在互为主备的部署模式下，mysql重启后会删除内存表，就会往binlog里面写入`delete from t1`，当这条语句传递给另一个master，就会把它自己库上的内存表t1删除，就会出现莫名其妙地，一个主库的内存表突然就被清空了！

## 3.4.自增主键

在mysql中，要使用自增主键的关键字是`AUTO_INCREMENT`，但是它只能保证**递增**，但是无法保证**连续递增**！！！自增值不会保存在表里，不同的存储引擎对于自增至的保存策略不同：

- MyISAM引擎的自增值保存在数据文件中；
- InnoDB引擎的自增值在MySQL8之前，都是保存在内存中（每次重启第一次打开表时，就去找max(id)，然后加1）；在之后的版本里，将自增值保存在redo log中；

**自增值修改机制**

1. 如果插入数据时主键字段为0、null或未指定，那么就将这张表的当前自增值填充到此次插入的数据中
2. 如果插入数据时主键字段有具体值X，当前自增值Y。若X<Y，自增值不变；若X &ge; Y，生成新的自增值。生成算法为：auto_increment_offset 和 auto_increment_increment是两个系统参数，分别用来表示自增的初始值和步长，默认值都是1。从auto_increment_offset开始，以auto_increment_increment为步长，持续叠加，直到找到第一个大于X的值，作为新的自增值。

**自增值不连续原因**

1. 唯一键冲突。因为自增值的生成，是在真正执行插入数据之前，如果语句插入的时候碰到唯一键冲突会执行失败，但是mysql不会把自增值还原，所以下一次语句插入的时候，就会拿到新的自增值，导致不连续发生；

2. 事务回滚。同上面一样，事务执行期间，自增值就已经生成，但是在事务回滚的时候，不会在还原自增值；

3. 批量申请自增id。mysql对申请自增主键id做了优化，对于同一个语句去申请自增id时，每次申请到的自增主键id个数是上一次的两倍。比如说：

   ```sql
   -- 先对表t插入4条数据
   insert into t values(null, 1,1);
   insert into t values(null, 2,2);
   insert into t values(null, 3,3);
   insert into t values(null, 4,4);
   
   -- 批量拷贝表t的数据到表t2
   create table t2 like t;
   insert into t2(c,d) select c,d from t;
   insert into t2 values(null, 5,5);
   
   -- insert…select实际上往表t2中插入了4行数据, 但这四行数据是分三次申请的自增id, 第一次申请到了id=1, 第二次被分配了id=2和id=3, 第三次被分配到id=4到id=7. 由于这条语句实际只用上了4个id, 所以id=5到id=7就被浪费掉了. 之后执行insert into t2 values(null, 5,5)语句时, 实际上插入的数据就是（8,5,5) 很明显自增主键的连续性就被破坏了
   ```

**自增锁优化**

不同事务在申请自增主键时，为了防止主键重复问题，肯定需要对自增值加锁，称为“自增id锁”。那么这个锁的释放时机是怎么样的？MySQL 5.1.22版本新增参数`innodb_autoinc_lock_mode`，用来控制自增锁的释放：

1. **值为0**，表示自增锁的范围是语句级别，即语句执行结束后才释放锁，当然这样会影响并发度
2. **值为1**（默认值），分为不同的情况：
   - 普通insert语句，自增锁在申请之后就立即释放；
   - 批量插入语句，例如：insert...select、replace...select、load data等，自增锁要等到语句结束后才释放；之所以要这样区分，是因为执行这种批量插入语句时，mysql预先无法知道要申请多少个自增id。

3. **值为2**，每次申请到自增值后就释放锁；

# 4.【主从复制】

mysql集群的负载均衡，读写分离和高可用都是基于复制实现。它的复制机制分为：异步复制、半同步复制和并行复制。

- **异步复制**

  异步复制是mysql自带的最原始的复制方式，主库和备库成功建立起复制关系后，在备库上会有一个IO线程去主库拉取binlog，并将binlog写到本地，就是下图中的Relay log，然后备库会开启另外一个SQL线程去读取回放Relay log，通过这种方式达到Master-Slave数据同步的目的。

- **半同步复制**

  异步复制会产生主从延迟问题，半同步复制就是为了解决数据一致性而产生的。理解啥是半同步复制，可以先了解下同步复制：一个事务在Master和Slave都执行后，才返回给用户执行成功（其实就是2PC协议）；MySQL只实现了本地redo-log和binlog的2PC，Slave在接收到日志后就响应Master（数据还未执行），这种就称为半同步复制。目前实现半同步复制主要有两种模式，AFTER_SYNC模式和AFTER_COMMIT模式。两种方式的主要区别在于是否在存储引擎提交后等待Slave的ACK。

- **并行复制**

  半同步复制可以解决数据一致性的问题，但是性能变低了，Master产生binlog的速度远远大于Slave SQL线程消费的速度，照样产生主从延迟。所以需要让Slave并行复制，可以IO线程并行，也可以SQL线程并行。并行IO线程，可以将从Master拉取和写Relay log分为两个线程；并行SQL线程则可以根据需要做到库级并行，表级并行，事务级并行。库级并行在mysql官方版本5.6已经实现

  SQL并发复制需要保证事务有序进行。Slave必需保证回放的顺序与Master上事务执行顺序一致，因此只要做到顺序读取binlog，将不冲突的事务并发执行即可。对于库级并发而言，协调线程（coordinator）要保证执行同一个库的事务放在一个工作线程串行执行；对于表级并发而言，协调线程要保证同一个表的事务串行执行；对于事务级而言，则是保证操作同一行的事务串行执行。协调线程在分发的时候，需要满足以下这两个基本要求：
  
  1. 不能造成更新覆盖。要求更新同一行的两个事务，必须被分发到同一个worker中。
  2. 同一个事务不能被拆开，必须放到同一个worker中。

## 4.1.基本原理

slave（从机）会从master（主机）读取biglog进行数据同步，mysql复制过程分为3步：

1. master将改变记录到二进制日志（binary log），这些记录称为二进制日志事件

2. slave将master的binary log拷贝到它的中继日志relay log

3. slave重做中继日志中的事件，将改变应用到自己的数据库中

<img src="./images/mysql主从流程.png" style="zoom:80%;" />

## 4.2.基本配置

一主一从常见配置，以window下的MySQL为主机，配置文件为my.ini；以Linux下的MySQL为从机，配置文件为my.cnf ，准备工作为：

- 主机和从机的Mysql版本要一致

- 主机和从机都配置在相应配置文件的`[mysqld]`结点下，都是小写

- 主机和从机相互ping通

### 4.2.1.配置

- master

  ```ini
  # 指定服务器唯一id, 范围[1,32], 必须配置
  server_id=1 
  
  # 开启bin-log功能
  log_bin=mysql-log-bin
  
  # 选择 ROW 模式
  binlog_format=ROW 
  
  # 指定要同步的数据库, 建议加上(如果省略，默认操作整个mysql)
  binlog_do_db=sym_test
  
  #指定不要同步的数据库，如果指定了binlog-do-db就不用再指定该项
  #binlog_ignore_db=mysql
  
  #是否只读，主机读写都可以
  read-only=0
  ```

- slave

  ```ini
  从机是放在linux系统下，所以配置文件是my.cnf
  
  # 指定服务器唯一id, 范围[1,32], 必须配置
  server_id=2
  
  # 开启中继日志
  relay_log=relay-log  
  
  # 指定要同步的数据库
  replicate_do_db=sym_test
  
  #指定不要同步的数据库
  #replicate_ignore_db=mysql  
  
  #是否只读，从机一般只读
  read-only=1
  ```

### 4.2.2.启动

- master

  ```sql
  -- 为从库生成账户: salve/slave123
  grant replication slave on *.* to 'slave'@'127.0.0.1' identified by 'slave123' ;
  
  -- 刷新权限
  flush privileges;
  
  -- 查看权限
  select * from mysql.user;
  
  -- 查看binlog记录，后面从库需要根据这个配置开启复制
  show master status;
  ```

  1. 主从配置文件改过以后，都重启后台mysql服务;

  2. 主从都关闭防火墙；（window手动关闭，linux用命令：service iptables stop）

  3. 主机上建立账户并授权slave

     grant replication slave on *.* to【账户名】@【从机IP】 identified by '密码';

     (从机如果很多，可以直接用”%”替代，表示任意ip)，如：

     ```sql
     grant replication slave on *.* to 'zhangsan'@ '192.168.1.1' identified by '123';
     ```

  4. 查询master状态，记录File和position的值

     ```sql
     show master status;
     ```


- slave

  1. 从机重新启动后需要连接到主机上，通过执行sql：

  ```sql
  change master to master_host='主机ip',master_port='主机端口',
  master_user='主机授权的用户名',master_password='密码',
  master_log_file='mysqlbin.具体数字',
  master_log_pos=具体值;
  ```

    (需要参照上面的第3、第4步) 如：

  ```sql
  change master to master_host='127.0.0.1',master_port=3308,
  master_user='slave',master_password='slave123', 
  master_log_file='mysql-log-bin.000035',
  master_log_pos=341;
  ```

  2、启动从机Mysql上的复制功能

   ```sql
   start slave;
   ```

  3、查询从机状态，

  ```sql
  show slave status;
  ```

    如果下面2个参数值都为yes，则说明配置正确：

  ```tex
  slave_io_running:yes
  slave_sql_running:yes
  ```

### 4.2.3.停止

在从机mysql上执行sql：

```sql
stop slave;
```