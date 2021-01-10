<center><h1>kafka</h1></center>
## 1、安装单机版的kafka

### 1.1、下载kafka压缩包

下载地址在：https://mirror-hk.koddos.net/apache/kafka/2.7.0/kafka_2.12-2.7.0.tgz

解压（注意：我这里在windows下使用，用git base客户端进行操作）

![gitstash](D:\jeffchan\markdown\image\gitbase.png)

```shell
tar -xzf  kafka_2.12-2.7.0.tgz
```

### 1.2、启动kafka

#### 1.2.1、启动zookeeper

进入对应的目录，然后执行：bin/zookeeper-server-start.sh config/zookeeper.properties

打印的日志信息：

```shell
[2021-01-09 17:12:28,498] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
[2021-01-09 17:12:28,499] WARN config\zookeeper.properties is relative. Prepend .\ to indicate that you're sure! (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
[2021-01-09 17:12:28,500] WARN \tmp\zookeeper is relative. Prepend .\ to indicate that you're sure! (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
[2021-01-09 17:12:28,518] INFO clientPortAddress is 0.0.0.0:2181 (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
```



#### 1.2.2、启动kafka

执行指令： bin/kafka-server-start.sh config/server.properties

打印的日志：

```shell
[2021-01-09 17:18:30,404] INFO Registered kafka:type=kafka.Log4jController MBean (kafka.utils.Log4jControllerRegistration$)
[2021-01-09 17:18:30,858] INFO Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation (org.apache.zookeeper.common.X509Util)
[2021-01-09 17:18:30,895] INFO starting (kafka.server.KafkaServer)
[2021-01-09 17:18:30,896] INFO Connecting to zookeeper on localhost:2181 (kafka.server.KafkaServer)
```

#### 1.2.3、创建一个topic

执行指令：bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic jeffchan0

打印的日志：

```java
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
Created topic jeffchan0.
```

我在windows的环境下，这里有个异常：

<font color='red'>log4j:ERROR Could not read configuration file from URL [file:/d/jeffchan/other-meterial/kafka_2.12-2.7.0/bin/../config/tools-log4j.properties].
java.io.FileNotFoundException: \d\jeffchan\other-meterial\kafka_2.12-2.7.0\bin\..\config\tools-log4j.properties (???????????)</font>



解决方法：直接将复制config目录：tools-log4j.properties文件的绝对路路径（每个人的绝对路径一般不同，你复制你对应的绝对路径就可以了）：

D:\jeffchan\other-meterial\kafka_2.12-2.7.0\config\tools-log4j.properties

打开：kafka-run-class.sh文件，将：

$base_dir/config/tools-log4j.properties 特换成绝对路经 D:\jeffchan\other-meterial\kafka_2.12-2.7.0\config\tools-log4j.properties

然后重新启动kafka



查看上述创建的topic:  

#### 1.2.4、启动生产者

执行命令(往topic jeffchan0中发送消息)：

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic jeffchan0
```

启动之后，可以往主题 jeffchan0中发送消息

![topicjeffchan](D:\jeffchan\markdown\image\topicjeffchan.png)



#### 1.2.5、启动消费者

执行命令：注意--from-beginning表示从头开始读取

```shell
 bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic jeffchan0 --from-beginning
```

![consumer](D:\jeffchan\markdown\image\consumer.png)

可以发现，上述生产者发布的消息已经被消费者消费了。



## 2、kafka原理解析

kafka一般以集群进行工作，集群一般由多台机器组成，一台机器一般由一个kafka实例，就是一个broker

kafka中消息是以主题topic进行分类的，生产者和消费者对消息的操作都是面向topic的，topic是逻辑上的概。

一个topic会有多个分区partitiion，而partitiion是物理上的概念，一个partitiion对应可以对应多个log文件，log文件存储的就是消息数据，生产者不断往log文件末端追加内容，并且每一条数据都有相应的offset，该offset记录的维度是消费者组+topic+分区的，消费者组每个消费者组都会实时记住消费到了哪个offset,便于下次继续消费。

由于生产者会不断追加日志到log末尾，未防止log文件过大导致数据定位效率低下，kafka采用了分片和索引机制，每个partition分为多个segment段，每个段对应两个文件，一个.index，一个.log文件，这个文件都位在topic+分区序号的文件夹下。index和log文件以当前segment的第一条消息的offset命名，index存的是索引信息，log文件储存数据，索引文件中的数据指向数据文件中的数据的物理地址。

下来文件加就是topic + partition的目录：

![tp](D:\jeffchan\markdown\image\tp.png)

目录里面的内容：

![tpdata](D:\jeffchan\markdown\image\tpdata.png)

可以看到对应的文件为.index和.log，同时明明从offset等于0开始命名。这里.log文件的大小，可以通过：log.segment.bytes=xxx来设置

```java
# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=1073741824
```

就是当文件的大小超出了这个大小，就生成另外一个通过开始offset命名的文件log和index文件。



### 2.1、分区策略

#### 2.11、kafka分区的原因

如果只有一个分区，那么topic内的所有消息都只能存在一个broker,那么这个broker就会存在操作效率和空间是否充足的问题了，无法作到水平扩展，为了解决效率和空间存储不足问题，那么将一个topic中的数据分一部分到其他broker上就很自然了。如果一个topic划分成多个分区，数据分布在不同的broker上，那么就可以实现并发读取，写入，还可以根据数量的大小，及时调整分区数来横向拆分。

#### 2.12、kafka分区原则

1、在指明partition的情况下，直接将知名的值作为partition值。

2、在没有指名partition但又key的情况下，将key的hash值和topic的partition数进行区域得到partition值

3、在既没有partion也没有key的情况下，第一次调用的时候会随机生成一个值，然后在这个值的基础上每次递增，然后和可用的partition数取余得到partition的值



#### 2.13、数据的可靠性

为保证生产者发送的数据能可靠的发送到指定的topic，topic的每隔partition收到生产者发送的数据后，都需要向生产者发送ack，如果生产者收到ack，就会进行下一轮的发送，否者重新发送数据。

这里leader收到生产者的消息后，要确保follower也同步完成才发送ack，这样才能保证在leader宕机之后能从follower中选出新的leader。



多少个follower同步完成之后才发送ack?

1、半数以上的follower同步完成之后才发送ack

（优点是延迟低，但是选举新的leader是，容忍n台结点的故障，需要2n+1个副本。）

2、全部follower完成，才发送ack

（kafka采用这个方式，虽然这个方式延迟比较高，但影响相对较小，这种方式，容忍n台节点故障，需要n+1个副本）



ISR:采用了第二种方案后，leader接收到数据，所有follower都开始同步数据，但有一个follower，因为某种故障，不能进行同步，那么leader就需要一直等下去，直到它同步了才能发送ack,这个问题怎么解决？



leader维护一个动态的ISR(副本同步集合)，当ISR中的所有follower都同步完后，leader就会发送ack,如果有follower长时间没有同步，超过某个阈值之（replica.lag.time.max.ms指定）后，就会被踢出ISR,leader发生故障之后，会冲ISR中重新选举leader。



kafka应答ack有三种级别可以配置:

0： 生产者不用等broker的ack，可能会丢失数据

1：生产者等待broker的ack，分区的leader落盘成功后返回ack,如果在follower

同步之前leader故障了，那么将会丢失数据。

-1:  生产者等待broker的ack,分区在leader和follower全部落盘后才返回ack,但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成数据的重复。



故障处理方式：

1、HW: 高水位标记（所有最后一个offset的副本中最小的一个）

比如：A副本最后一个offset为10，B副本最后一个为18，C副本最后一个为21

那么这个HW=10

2、LEO指的是每个副本最大的offset



follower故障：follower发生故障后会被临时退出ISR,带follower恢复后，follower会读取本地磁盘上次的HW，并将log文件中高于HW的部分截取掉，从HW开始向leader进行同步。等follower的LEO待遇等于该分区的HW,就可以从新加入ISR.



leader故障

leader发生故障之后，会从ISR中选出一个新的leader，之后为了保证多个副本之间的一直型，其余的follower会将各自的log文件高于HW的那部分截掉。从新从leader同步数据。

（这个只能保证各个副本之间的一致性，并不能保证数据不丢失和不重复）



将ACK设置为-1，可以保证生产者和broker之间不会丢失数据，但是不能保证不重复

设置为0，可以保证生产者每条消息都会只发送一次，即最多一次，但是不保证不丢失。



精准发送：at least once + 幂等 = exactly once

要启用幂等性，只需要将producer的 enable. idempotence=true，kafka的幂等性实现其实就是将原来下游需要做到幂等的数据放在了数据上游，开启幂等性的生产者在初始化的时候会被分配一个PID,发往同一个分区的消息会附带sequence number,而broker端会对<PID, partition,seqnumber>做缓存，当具有相同的主键消息提交时，broker只会持久化一条。

但是如果重启，PID会变化，同时不同分区具有不同的主键，所以幂等性无法跨分区跨会话。



### 2.2、消费方式

consumer采用pull的模式从broker读取数据。为什么不采用push，因为push模式很难适应不同的消费者。因为发送数据是由生产者发送的，kafka的目标是尽可能快发送数据，这样容易造成消费者处理不过来，出现网络堵塞，服务负载高导致无法响应等问题。

通过pull，消费者可以根据自己接受的速度来消费数据，但是不足之处就是当没有数据可以消费时，消费者也需要不停的轮询，浪费资源。针对这一点，消费者在消费数据时，会传入一个时长参数timeout,如果当前没有数据返回，消费者会等待timeout时间再去消费。



### 2.3、分区分配策略

分区分配策略是指哪个分区由哪个消费者消费，kafka有两种分配方式：RoundRobin，Range，Sticky



Range: 这种分配方式，是针对主题维度的，就是比如说有两个主题topicA, topicB，都具有三个分区，如果都被消费者组A（两个消费者a，b）进行消费，那么会分别对主题里的分区进行分配，如果不能整除，会出现一个消费者多读取。比如这里a分了两个topicA分区，b分了一个topicA分区。然后topicB的情况也是，所以这种方式就会出现消费者分配分区数不均衡的问题，订阅的主题越多，这个不均衡就越大。



RoundRobin：这种方式解决了上述Range方式中那个问题，就是它会将所有主题的分区进行一个排序，然后对消费者进行一次排次，然后进行分配。就比如上述的场景中，消费者都是订阅相同的主题，那么这种方式各个消费者之间分区的差别不超过一个。但是如果消费者订阅的主题不是相同的情况下，会出现订阅分配不均的情况。

要使用这种算法需要满足条件：所有消费者订阅的主题相同同时开启的消费线程也相同



Sticky：