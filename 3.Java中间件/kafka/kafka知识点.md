# 1.kafka介绍

Kafka 是消息引擎系统，也是分布式流处理平台。

![](./images/kafka架构.png)

## 1.1.基本概念

- **Producer**：生产者即数据的发布者，该角色将消息发布到kafka的topic中。broker将该消息追加到前档用于追加数据的segment中。生产者发送的消息，储存到一个partition中，生产者也可以指定数据储存的partition
- **Consumer**：消费者可以从broker中读取数据。消费者可以消费多个topic中的数据
- **Topic**：在kafka中使用一个类别属性来划分数据的所属类，划分数据的这个类称为topic。如果把kafka看做一个数据库，topic可以理解为数据库中的一张表，topic面子即为表名
- **Partition**：topic中的数据分割为一个或多个partition（分区）。每个topic至少有一个partition，每个partition中的数据使用多个segment文件存储。partition中的数据是有序的，partition间的数据丢失了数据的顺序。如果topic有多个partition，消费数据时候就不能保证数据的顺序。在需要严格保证消息的消费顺序的场景下，需要将partition数据设置为1；
- **Partition offset**：每条消息都有一个当前Partition下唯一的64字节的offset，它指明了这条消息的起始位置
- **Replicas of partition**：分区副本，每个分区partition下可以配置若干个副本，其中只能有 1 个领导者副本和 N-1 个追随者副本。追随者副本不会被消费者消费，它只用于防止数据丢失，即消费者不从为follower的partition中消费数据，而是从leader的partition中读取数据。副本之间是一主多从的关系
- **Broker**：kafka集群包含一个或多个服务器，服务器节点称为broker。broker存储topic的数据，如果某topic有N个partition，集群有N个broker，那么每个broker储存该topic的一个partition。如果某topic有N个partition，集群有（N+M）个broker，那么其中有N个broker储存该topic的一个partition，剩下的M歌broker不存储该topic的partition数据。如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一个或多个partition。在实际生产环境中，尽量避免这种情况的发生，这种情况容易导致kafka集群数据不均衡
- **Leader**：每个partition有多个副本，其中有且仅有一个作为Leader，Leader是当前负责数据的读写的partition
- **Follower**：Follower跟随Leader，所有写请求都通过Leader路由，数据变更会黄渤给所有Follower，Follower与Leader保持数据同步。如果Leader失效，则从Follower中选举出一个新的Leader。当前Follower与Leader挂掉、卡主或者同步太慢，leader会把这个Follower从“in sync replicas”（ISR）列表中删除，重新创建一个Follower
- **Zookeeper**：Zookeeper负责维护和协调broker。当kafka系统中新增了broker或者某个Broker发生故障失效时，由zookeeper通知生产者和消费者。生产者和消费者依据zookeeper的broker状态信息与broker协调数据的发布和订阅任务
- **AR**（Assigned Replicas）：分区partition中所有的副本统称为AR
- **ISR**（In Sync Replicas）：所有于Leader部分保持一定程度的副本组成ISR
- **OSR**（Out of Sync Replicas）：与Leader副本同步滞后过多的副本
- **HW**（High Watermark）：高水位，标识了一个特定的offset，消费者只能拉取到这个offset之前的消息
- **LEO**（Log End Offset）：即日志末端位移(log end offset)，记录了该副本底层日志(log)中下一条消息的位移值。注意是下一条消息！也就是说，如果LEO=10，那么标识该副本保存了10条消息，位移值范围是[0,9]

## 1.2.使用场景

1. **日志收集**：kafka可以收集企业级微服务的日志信息，通过统一接口服务的方式开放给各种消费者，诸如Hadoop、Hbase、Solr和Elasticsearch等
2. **消息系统**：解耦和生产者和消费者、缓存消息等
3. **用户活动跟踪**：kafka长用于记录web用户或者app用户的各种活动，例如浏览网页、搜索、点击等活动，这些活动信息发布到kafka的topic中，然后消费者通过订阅这些topic做实时的监控分析，或者装载到Hadoop、数据仓库中做离线分析和挖掘
4. **运营指标**：kafka常用于记录运营监控数据，包括手机各种分布式应用的数据，生产各种操作的几种反馈，比如报警和报告
5. **流式处理**：比如spark streaming和storm

## 1.3.技术优势

- **可伸缩性**：kafka的两个重要特性造就了它的可伸缩性
  1. kafka集群可以在运行期间轻松地扩展或收缩，而不会宕机
  2. 可以扩展一个kafka主题来包含更多的分区，由于一个分区无法扩展到多个代理所以它的容量受到代理磁盘空间的限制。能够增加分区和代理的数量意味着单个主题可以储存的数据量是没有限制的

- **容错性和可靠性**：kafka的设计方式使某个代理的故障能够被集群中的其它代理检测到，由于每个主题都可以在多个代理上复制，所以集群可以在不中断服务的情况下从此类故障中恢复并继续运行
- **吞吐量**：代理能够以超快的速度有效地储存和检索数据

# 2.生产者Producer

kafka消息的发送方

## 2.1.基本组件

- 序列化器

消息要在网络上传输，必须以字节流的形式，即需要被序列化。kafka的序列化器接口：`org.apache.kafka.common.serialization.Serializer`，默认提供了字符串序列化器、整型序列化器和字节数组序列化器等等..

- 分区器

分区器是用来决定消息要发送到broker的哪个分区上，kafka的分区器接口：`org.apache.kafka.clients.producer.Partitioner`，若用户没指定分区器实现，kafka会使用`org.apache.kafka.clients.producer.internals.DefaultPartitioner`，它是根据传递消息的key来进行分区的分配，即hash(key)%numPartitions，如果key相同的话就会分配到统一分区

- 拦截器

拦截器是用来对生产者做定制化的逻辑控制，可以在消息发送之前进行额外的处理。kafka的拦截器接口：`org.apache.kafka.clients.producer.ProducerInterceptor`。一般用于以下场景：

​	①按照某个规则过滤掉不符合要求的消息

​	②修改消息的内容

​	③统计类需求

## 2.2.分区机制

分区partition的作用就是提供负载均衡的能力，实现系统的高伸缩性（Scalability）。这个概念在分布式系统中很常见，只不过叫法可能不同。在 Kafka 中叫分区，在 MongoDB 和 Elasticsearch 中就叫分片 Shard，而在 HBase 中则叫 Region，在 Cassandra 中又被称作 vnode，底层的分区思想是一样。

实际开发中，生产者分区机制，除了支持负载均衡以外，还会有一些业务上的需求，比如规定带有指定值的消息只能发送到规定的partition上。这就需要使用到kafka生产者的分区策略， **所谓分区策略是决定生产者将消息发送到哪个分区的算法。**Kafka 提供了默认的分区策略，同时也支持自定义分区策略，就是实现[基本组件](# 2.1.基本组件)中的分区器接口

- 轮询策略

Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0。这是kafka默认的生产者分区策略

![](./images/生产者分区-轮询策略.png)

- 随机策略

Randomness 策略。所谓随机就是随意地将消息放置到任意一个分区上。**如果追求数据的均匀分布，还是使用轮询策略比较好**。事实上，随机策略是老版本生产者使用的分区策略，在新版本中已经改为轮询了。

![](./images/生产者分区-随机策略.png)

- 消息键保序策略

 Key-ordering 策略。Kafka 允许为每条消息定义消息键，简称为Key，开发中可以为Key附上实际业务属性，这样可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略

![](./images/生产者分区-消息建保序策略.png)

## 2.3.消息发送流程

![](./images/kafka-producer消息发送流程.png)

# 3.消费者Consumer

## 3.1.消费者和消费组

kafka消费者可以加入一个消费组，一个消费组可以监听多个topic。kafka会保证一个消费组内的消费者会各自承担一个topic的分区消费（不会出现一个partition由同一个消费组内的两个消费者同时消费.）

## 3.2.消息接收

### 3.2.1.位移提交

对于kafka而言，它的每条消息都有唯一的offset，表示消息在分区中的位置。当消息从broker返回消费者时，broker并不会跟踪消息是否被消费者消费到，而是让消费者自身来管理消费的位移，并向消费者提供更新位移的接口，这种更新位移的方式成为提交(commit)，注意kafka只保证消息在一个分区的顺序性

- 自动提交
- 同步提交
- 异步提交

### 3.2.2.指定位移

### 3.2.3.再均衡

再均衡，指的是在kafka consumer所订阅的topic发生变化时发生的一种分区partition重分配机制，一般有三种情况：

- consumer group新增或者删除某个consumer，导致其所消费的分区需要分配到组内其他的consumer上；
- consumer订阅的topic发生变化，比如订阅的topic采用的是正则表达式的形式，如`test-*`。此时如果有一个新建了一个topic `test-user`，那么这个topic的所有分区也是会自动分配给当前的consumer的，此时就会发生再平衡；
- consumer所订阅的topic发生了新增分区的行为，那么新增的分区就会分配给当前的consumer，此时就会触发再平衡

kafka提供的再平衡策略主要有三种：`Round Robin`，`Range`和`Sticky`，默认使用的是`Range`。这三种分配策略的主要区别在于：

- `Round Robin`：会采用轮询的方式将当前所有的分区依次分配给所有的consumer；
- `Range`：首先会计算每个consumer可以消费的分区个数，然后按照顺序将指定个数范围的分区分配给各个consumer；
- `Sticky`：这种分区策略是最新版本中新增的一种策略，其主要实现了两个目的：
  - 将现有的分区尽可能均衡的分配给各个consumer，存在此目的的原因在于`Round Robin`和`Range`分配策略实际上都会导致某几个consumer承载过多的分区，从而导致消费压力不均衡；
  - 如果发生再平衡，那么重新分配之后在前一点的基础上会尽力保证当前未宕机的consumer所消费的分区不会被分配给其他的consumer上；

# 4.主题Topic

# 5.分区Partition

kafka将主题划分为多个分区（partition），会根据分区规则选择把消息存储到哪个分区中。如果分区规则设置的合理，那么所有的消息将会被均匀地分布到不同的分区中，这样就实现了负载均衡和水平扩展。另外，多个订阅者可以从一个或者多个分区中同时消费数据，以支撑海量数据处理能力。注：由于消息时以追加到分区中，多个分区顺序写磁盘的总效率比随机写内存还要高，是kafka高吞吐量的重要保障之一

## 5.1.副本机制

Kafka 0.8 以后，提供了 HA 机制，就是 replica（复制品） 副本机制。一个主题可以有多个分区，例如下图，红色部分：topic1-part0、topic1-part1、topic1-part2，表示主题topic1分为了3个分区；同时每个分区都有2个副本（绿色部分），这些副本保存在不同的broker上（分区和它的副本最好不要保存在同一个节点上），在一个broker出错时，leader在这台broker上的分区会变得不可用，kafka会自动移除leader，再从其它副本中选一个作为新的leader。

![](./images/kafka副本机制.png)

## 5.2.分区分配策略

kafka默认的消费逻辑设定，一个分区只能被同一个消费组内的一个消费者消费，如果消费者过多，出现了消费者的数量大于分区的数量，那么就会有消费者分配不到任何分区 。kafka提供了消费者客户端参数partition.assignment.strategy用来设置消费者与订阅主题之间的分配策略，默认情况下，此参数的值为`org.apache.kafka.clients.consumer.RangeAssignor`。kafka还提供了另外两种分配策略：`org.apache.kafka.clients.consumer.RoundRobinAssignor`和`org.apache.kafka.clients.consumer.StickyAssignor`。kafka允许这只多个分配策略，彼此之间用逗号分隔

### 5.2.1.RangeAssignor

RangeAssignor策略的原理是按照消费者总数和分区总数进行整除运算来获得一个跨度，然后将分区按照跨度进行平均分配。对于每一个topic，RangeAssignor策略会将消费组内所有订阅这个topic的消费者按照名字的字典序排序，然后为每个消费者划分固定的分区范围，如果不够平均分配，那么字典序靠前的消费者会多分配一个分区。

假设n=分区数/消费者数，m=分区数%消费者数，那么前m个消费者每个分配n+1个分区，后面的（消费者数量-m）个消费者每个分配n个分区

### 5.2.2.RoundRobinAssignor

RoundRobinAssignor策略的原理是将消费组内所有消费者以及消费者所订阅的所有topic按照partition按照字典序排序，然后通过轮询方式组个将分区依次分配给每个消费者。

假设消费组中有2个消费者C0和C1，都订阅了主题t0和t1，并且每个主题都有3个分区，那么所订阅的所有分区可以标识为：t0p0，t0p1，t0p2，t1p0，t1p1，t1p2。最终的分配结果

```tex
消费者C0：t0p0，t0p2，t1p1
消费者C1：t0p1，t1p0，t1p2
```

但是同一个消费组内的消费者所订阅的信息是不同的，就会出现分配不均匀的现象，如果某个消费者没有订阅消费组内的某个topic，那么在分配分区的时候此消费者将分配不到这个topic的任何分区。例如：某个消费组内的有3个消费者C0、C1和C2，消费组订阅了t0、t1和t2，这3个主题分别有1、2、3个分区，即整个消费组订阅了t0p0、t1p0、t1p1、t2p0、t2p1、t2p3共6个分区。而且，消费者C0订阅的是t0主题，C1订阅t0和t1主题，C2订阅t0、t1和t2，那么最终结果为：

```tex
消费者C0：t0p0
消费者C1：t1p0
消费者C2：t1p1、t2p0、t2p1、t2p2
```

其实可以吧分区t1p1分配给C1消费，减轻C2的研力

### 5.2.3.StickyAssignor

StickyAssignor策略是从kafka 0.11x版本开始引入，它有两个目的：

- 分区的分配要尽可能地均匀
- 分区的分配要尽可能的与上次分配的保持相同

如果上面两者发生冲突，第一个目标优先于第二个目标

# 6.存储结构

每一个partition相当于一个巨型文件，被平均分配到多个大小相等的segment（段）数据文件里。但每一个segment file消息数量不一定相等，这样的特性方便old segment file高速被删除，默认情况下每一个segment file文件大小为1G。partition仅仅只要支持顺序读写即可，segment文件生命周期由服务器配置参数决定。

segment file由两大部分组成：index file和data file，后缀`.index`和`.log`分别表示sgement索引文件、数据文件

## 6.1.日志索引

### 6.1.1.数据文件分段

比如说有100条message，它们的offset是从0~99，假设将数据分为5段，依次为0-19,20-39,40-59...以此类推，每段放在一个单独的数据文件里面，数据文件以该段中最小的offset命名。这样再查找指定offset的Message时，用二分查找就可以定位到该Message在哪个段中

### 6.1.2.偏移量索引

在数据文件分段的基础上，kafka为每个分段后的数据文件建立了索引文件，文件名与数据文件的名字是一样的，只是文件扩展名为`.index`。

![](./images/kafka偏移量索引.png)

比如要查找offset为7的Message，

- 首先是用二分查找确定他是在哪个LogSegment中，自然是第一个Segment中；
- 打开这个Segment的index文件，用二分查找找到offset小于或等于指定offset的索引条目中最大的那个offset；
- 自然offset为6的索引就是要找的项，通过索引文件可得offset为6的Message在数据文件中的位置为9807

这套机制是建立在offset是有序的，索引文件被映射到内存中，所以查找的速度还是很快的。总而言之，kafka的消息存储采用了分区（partition）、分段（LogSegment）和稀疏索引等方式以实现高效性！

## 6.2.日志清理

### 6.2.1.日志删除

kafka日志管理器允许订制删除策略，默认策略是删除修改时间N天之前的日志（也就是按时间删除），也可以使用另外一个策略，保留最后的N GB数据的策略（按大小删除）。同时，为了避免在删除的时候阻塞读操作，采用了copy-on-write形式的实现，删除操作进行时，读取操作的二分查找功能实际是在一个静态的快照副本上进行，这类似于java的CopyOnWriteArrayList

```properties
#启用删除策略
log.cleanup.policy=delete
# 超过指定时间清理
log.retention.hours=16
# 超过指定大小后，删除旧的信息
log.retention.bytes=1073741824
```

### 6.2.2.日志压缩

将数据压缩，只保留每个key最后一个版本的数据。首先在broker的配置中设置`log.cleaner.enbale=true`启用cleaner，这个默认是关闭的。在Topic的配置中设置`log.cleanup.policy=compact`启用压缩策略

## 6.3.磁盘存储

kafka实现高吞吐量的存储原因：

- 消息顺序追加
- 页缓存
- 零拷贝

### 6.3.1消息顺序追加

kafka在设计的时候，采用文件追加的方式来写入消息，即只能在日志文件的尾部追加新的消息，并且不允许修改已经写入的消息，这种方式属于典型的顺序写入操作。

### 6.3.2.页缓存

kafka中大量使用页缓存，也是kafka实现高吞吐量的重要因素之一

### 6.3.3.零拷贝

kafka零拷贝技术只用将磁盘文件的数据复制到页面缓存一次，就可以将数据从页面缓存直接发送到网络中（发送给不同的订阅者时，都可以使用同一个页面缓存），避免重复操作。例如：假设有10个消费者，传统方式下，数据复制次数为4*10=40次，而使用零拷贝技术，只要1+10=11次，一次为从磁盘复制到页面缓存，10次表示10个消费者各自读取一次页面缓存。

# 7.稳定性

## 7.1.幂等性

生产者才发生消息给broker，期间如果发生网络异常，生产者由于没有收到broker的ack响应，会重新发送消息。在生产者进行重试的时候，就有可能会重复写入消息，kafka的幂等性可以避免这种情况，但是kafka的幂等性具有一定的限制性：

- 会话级别的幂等性，kafka只能保证producer在单个会话内不丢不重，如果producer出现宕机再重启是无法保证幂等的（无法获取之前的幂等状态信息）、
- 幂等性不能跨多个Topic-partition，只能保证单个partition内的幂等性，当涉及多个Topic-partition时，这中间的状态并没有同步

producer使用幂等性的示例非常简单，只需要把producer的配置`enable.udempotence`设置为true即可

## 7.2.事务

幂等性并不能跨多个分区运作，而事务可以弥补这个缺憾，事务可以保证对多个分区写入操作的原子性。操作的原子性是指多个操作要么全部成功，要么全部失败。为了实现事务，应用程序必须提供唯一的`transactionalId`，而且要求生产者开启幂等性特性，所以必须设定：

```java
// 开启幂等性 + 指定事务ID
properties.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
properties.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "11232321");
```

```java
// 初始化事务
producer.initTransactions();
// 开启事务
producer.beginTransaction();
try {
    // doSomething
    producer.send(null);
    // 提交事务
    producer.commitTransaction();
} catch (Exception e) {
    // 回滚事务
    producer.abortTransaction();
}
```

## 7.3.控制器

在kafka集群中会有一个或者多个broker，其中有一个broker会被选举为控制器（kafka Controller），它负责管理整个集群中所有分区和副本的状态。当某个分区的leader副本出现故障时，由控制器负责为该分区选举新的leader副本。当检测到某个分区的ISR集合发送变化时，由控制器负责通知所有broker更新其元数据信息。当使用kafka-topics.sh脚本为某个topic增加分区数量时，同样还是由控制器负责分区的重新分配

kafka中的控制器选举的工作依赖于zookeeper，成功竞选为控制器的broker会在zookeeper中创建/controller这个临时节点。zookeeper还有一个与控制器相关的/controller_epoch节点，这个节点是持久节点，节点中存放的是一个整型的controller_epoch值。controller_epoch用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，也可以称为“控制器的纪元”

## 7.4.一致性保证

### 7.4.1.HW截断机制

在leader宕机后，控制器从ISR列表中选取新的leader，新的leader并不能保证已经完全同步了之前leader的所有数据，只能保证HW之前的数据是同步过的，此时所有的follower都要讲数据截断到HW的位置，再和新的leader同步数据，来保证数据一致。当宕机的leader恢复，发现新的leader种的数据和自己持有的数据不一样，此时宕机的leader会将自己的数据截断到宕机之前的HW位置，然后同步新leader的数据。

### 7.4.2.数据丢失场景

场景1：

![](./images/kafka水位-数据丢失-情况1.png)

上图中有两个副本：A和B，其中A是leader。假设producer端`min.insync.replicas=1`，当producer发送两条消息给A后，A写入到底层log，此时Kafka会通知producer说这两条消息写入成功。

但是在broker端，leader和follower底层的log虽都写入了2条消息且leader HW已经被更新到1，但follower HW尚未被更新（也就是上面紫色颜色标记的第二步尚未执行）。倘若此时副本B所在的broker宕机，那么重启回来后B会自动把LEO调整到之前的HW值，故副本B会做日志截断(log truncation)，将offset = 1的那条消息从log中删除，并调整LEO = 1，此时follower副本底层log中就只有一条消息，即offset = 0的消息。

B重启之后需要给A发FETCH请求，但若A所在broker机器在此时宕机，那么Kafka会令B成为新的leader，而当A重启回来后也会执行日志截断，将HW调整回0。这样，位移=1的消息就从两个副本的log中被删除，即永远地丢失了

场景2：

![](./images/kafka水位-数据丢失-情况2.png)

A依然是leader，A的log写入了2条消息，但B的log只写入了1条消息。分区HW更新到1，但B的HW还是0，同时producer端的`min.insync.replicas = 1`

如果A和B所在机器同时挂掉，然后假设B先重启回来，因此成为leader，分区HW = 0。假设此时producer发送了第3条消息(绿色框表示)给B，于是B的log中offset = 1的消息变成了绿色框表示的消息，同时分区HW更新到1。之后A重启回来，需要执行日志截断，但发现此时分区HW=1而A之前的HW值也是1，故不做任何调整。此后A和B将以这种状态继续正常工作

### 7.4.3.leader epoch

上面两种场景的根本原因在于HW值被用于衡量副本备份的成功与否以及出现failure时作为日志截断的一句，但是HW值的更新是异步的，尤其是需要FETCH请求处理流程才能更新，故期间如果发生崩溃都可能导致HW值过期。所以，kafka 0.11引入了leader epoch来取代HW值，所谓leader epoch就是基上一对值— `(epoch,offset)`，`epoch`表示leader的版本号，从0开始，当leader变更过1次后`epoch`就会+1，而`offset`对应于该`epoch`版本的leader写入第一条消息的位移。例如：

(0, 0)

(1, 120)

表示第一个leader从位移0开始写入消息，共写了120条[0,119]；而第二个leader版本号是1，从位移120处开始写入消息。broker会保存这样一个缓存，并定期写入一个checkpoint文件中，避免文件丢失

针对场景1：

![](./images/kafka水位-数据丢失-情况1-解决.png)

针对场景2：

![](./images/kafka水位-数据丢失-情况2-解决.png)