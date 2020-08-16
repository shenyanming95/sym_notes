# 1.集群配置

搭建zookeeper集群，建议至少使用3个机器节点（当然单机创建3个zk实例也不是不可以），最主要是需要进行不同的配置zoo.cfg，例如：

```properties
# server1
tickTime=2000
initLimit=5
syncLimit=2
dataDir=/opt/zookeeper/server1/data
dataLogDir=/opt/zookeeper/server1/dataLog
clientPort=2181
# server.{myid}={ip}:{leader服务器交换信息的端口}:{当leader服务器挂了后, 选举leader的端口}
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```



```properties
# server2
tickTime=2000
initLimit=5
syncLimit=2
dataDir=/opt/zookeeper/server2/data
dataLogDir=/opt/zookeeper/server2/dataLog
clientPort=2182
# server.{myid}={ip}:{leader服务器交换信息的端口}:{当leader服务器挂了后, 选举leader的端口}
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```



```properties
# server3
tickTime=2000
initLimit=5
syncLimit=2
dataDir=/opt/zookeeper/server3/data
dataLogDir=/opt/zookeeper/server3/dataLog
clientPort=2183
# server.{myid}={ip}:{leader服务器交换信息的端口}:{当leader服务器挂了后, 选举leader的端口}
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

然后依次启动这些服务器即可。

# 2.zookeeper配置

ZooKeeper 的配置项在`zoo.cfg` 配置文件中配置, 另外有些配置项可以通过 Java 系统属性来进行配置。  

## 2.1.启动配置

- clientPort：对客户端提供服务的端口；
- dataDir：保存快照文件的目录，若没有设置 dataLogDir ，事务日志文件也会保存到该目录下；
- dataLogDir：保存事务日志文件的目录，ZooKeeper 在提交一个事务之前，会先保证事务日志记录的落盘；

- tickTime：通信心跳数，Zookeeper服务器与客户端心跳时间，单位毫秒；也就是每个tickTime时间就会发送一个心跳，时间单位为毫秒。它用于心跳机制，并且设置最小的session超时时间为两倍心跳时间。(session的最小超时时间是2*tickTime)

- initLimit：LF初始通信时限。集群中的Follower跟随者服务器与Leader领导者服务器之间初始连接时能容忍的最多心跳数（tickTime的数量），用它来限定集群中的Zookeeper服务器连接到Leader的时限

- syncLimit：LF同步通信时限。集群中Leader与Follower之间的最大响应时间单位，假如响应超过syncLimit * tickTime，Leader认为Follwer死掉，从服务器列表中删除Follwer

- clientPort =2181：客户端连接端口，监听客户端连接的端口。