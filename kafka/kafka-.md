## 一. 回顾
producer->topic->consumer
集群运行，每个服务broker
tcp协议，Kafka提供java端，另一端可支持多种语言

**topic**
topic是对一组消息的归纳，每个topic分区
![](http://www.aboutyun.com/data/attachment/forum/201505/02/225851kqq1pnqbq81kblln.png)
每个分区都有个offset
consumer需要维护offset
分区目的：1、使得每个日志的数量不会太大；2、不同分区可单独操作，并发消费topic

**分布式**
每个分区，多个broker中有副本；
每个分区多个副本有一个leader，其他副本作为followers；
leader读写消息，followers则去复制leader；
如果leader down了，followers中的一台则会自动成为leader；
每个服务都会同时扮演两个角色：
- 自己分区的leader
- 其他分区的followers

**Producers**
将消息发送到topic中，指定分区；
可以指定分区函数，用户可以指定负载均衡算法；

**Consumers**
发布模式包括：queuing、publish-subscribe；
queuing模式：

```bash
consumers可以同时从服务器读取消息，但是每个消息只能被一个consumer得到
```
publish-subscribe模式：

```bash
Consumers加入consumer组，这个组内的consumer竞争这个消息，这是分组订阅；
所有的consumer在一个组中，是传统的queuing模式
每个consumer一个独立的组，是publish-subscribe模式
```
![](http://www.aboutyun.com/data/attachment/forum/201505/02/225852ng3ur3gmtc9v489o.png)

传统的队列在服务器上保存有序的消息，如果多个consumers同时从这个服务器消费消息，服务器就会以消息存储的顺序向consumer分发消息。虽然服务器按顺序发布消息，但是消息是被异步的分发到各consumer上，所以当消息到达时可能已经失去了原来的顺序，这意味着并发消费将导致顺序错乱。为了避免故障，这样的消息系统通常使用“专用consumer”的概念，其实就是只允许一个消费者消费消息，当然这就意味着失去了并发性。

通过分区的概念，Kafka可以在多个consumer组并发的情况下提供较好的有序性和负载均衡。将每个分区分只分发给一个consumer组，这样一个分区就只被这个组的一个consumer消费，就可以顺序的消费这个分区的消息。因为有多个分区，依然可以在多个consumer组之间进行负载均衡。注意consumer组的数量不能多于分区的数量，也就是有多少分区就允许多少并发消费。

Kafka只能保证一个分区之内消息的有序性，在不同的分区之间是不可以的，这已经可以满足大部分应用的需求。如果需要topic中所有消息的有序性，那就只能让这个topic只有一个分区，当然也就只有一个consumer组消费它。

## 二. Producer API
kafka.producer.SyncProducer
kafka.producer.async.AsyncProducer
实现了同一接口

```java
class Producer {
	//发送到指定分区
	publicvoid send(ProducerData<K,V> producerData);
	//批量发送	
	publicvoid send(List<ProducerData<K,V>> producerData);
	publicvoid close();
}
```
**本地缓存**
本地队列缓存，异步发送到broker;`producer.type=async`设置；
后台ProducerSendThread从队列中取数据，让EventHandler给handler发送消息
producer处理数据的不同阶段，可定制handler，添加日志追踪，监控等

**Encoder序列化消息**
默认是DefaultEncoder接口，可以实现这个接口

```java
interface Encoder<T> {
	public Message toMessage(T data);
}
```

**broker自动感知**
选中的broker失败了，会自动切换到下一个broker

**消息分区**
Partitioner类

```java
interface Partitioner<T> {
	int partition(T key, int numPartitions);
}
```
numPartitions:可用分区数量
默认的分区策略：`hash(key)%numPartitions`

## 三. Consumer API
两个级别：
1. 与指定broker保持连接，接收完消息关闭，无状态，每次读取都带有offset
2. 隐藏broker细节，利用条件指定topic，比如：黑白名单、正则表达式

级别1：

```java
class SimpleConsumer {
	//向一个broker发出读取请求
	public ByteBufferMessageSet fetch(FetchRequest request);
	//向一个broker发出读取请求,并得到一个相应集
	public MultiFetchResponse multifetch(List<FetchRequest> fetches);
	/**
	* 得到指定时间之前的offsets
	* 返回值是offsets列表，以倒序排序
	* @param time: 时间，毫秒,
	* 如果指定为OffsetRequest$.MODULE$.LATIEST_TIME(), 得到最新的offset.
	* 如果指定为OffsetRequest$.MODULE$.EARLIEST_TIME(),得到最老的offset.
	*/
	publiclong[] getOffsetsBefore(String topic, int partition, long time, int maxNumOffsets);
}
```
低级别为了一些对维持消费状态有特殊需求的场景，如Hadoop consumer这样的离线consumer

级别2：

```java
ConsumerConnector connector = Consumer.create(consumerConfig);
	interface ConsumerConnector {
	/**
	* 这个方法可以得到一个流的列表，每个流都是MessageAndMetadata的迭代，通过MessageAndMetadata可以拿到消息和其他的元数据（目前之后topic）
	* Input: a map of <topic, #streams>
	* Output: a map of <topic, list of message streams>
	*/
	public Map<String,List<KafkaStream>> createMessageStreams(Map<String,Int> topicCountMap);
	/**
	* 你也可以得到一个流的列表，它包含了符合TopicFiler的消息的迭代，
	* 一个TopicFilter是一个封装了白名单或黑名单的正则表达式。
	*/
	public List<KafkaStream> createMessageStreamsByFilter(TopicFilter topicFilter, int numStreams);
	/* 提交目前消费到的offset */
	public commitOffsets()
	/* 关闭连接 */
	public shutdown()
}
```
围绕KafkaStream展开，每个流代表一系列从一个或多个分区多和broker上汇聚来的消息，每个流由一个线程处理，可以在创建的时候通过参数指定想要的几个流；
一个流是多个分区多个broker的合并，但是每个分区的消息只会流向一个流。
`createMessageStreams`将consumer注册到topic上,这样consumer和brokers之间的负载均衡就会进行调整,会消耗更多时间；
API鼓励每次调用创建更多的topic流以减少这种调整。
createMessageStreamsByFilter方法注册监听可以感知新的符合filter的tipic。


## 四. 消息和日志
具有N个字节的消息的格式如下

```java
如果版本号是0
1. 1个字节的 "magic" 标记
2. 4个字节的CRC32校验码
3. N - 5个字节的具体信息

如果版本号是1
1. 1个字节的 "magic" 标记
2. 1个字节的参数允许标注一些附加的信息比如是否压缩了，解码类型等
3. 4个字节的CRC32校验码
4. N - 6 个字节的具体信息
```

my_topic日志，由my_topic_0和my_topic_1两个目录组成，目录中存放具体的数据文件；
数据文件：日志实体

```java
消息长度: 4 bytes (value: 1+4+n)
版本号: 1 byte
CRC校验码: 4 bytes
具体的消息: n bytes
```
每个消息都可以由一个64位的整数offset标注
offset标注了这条消息在发送到这个分区的消息流中的起始位置
每个日志文件的名称都是这个文件第一条日志的offset，第一个日志文件的名字就是00000000000.kafka
每相邻的两个文件名字的差就是一个数字S,S差不多就是配置文件中指定的日志文件的最大容量
消息的格式都由一个统一的接口维护，所以消息可以在producer,broker和consumer之间无缝的传递
![](http://www.aboutyun.com/data/attachment/forum/201505/03/000751s9xnzlj99b3j2rms.png)

**写操作**
消息被不断的追加到最后一个日志的末尾，当日志的大小达到一个指定的值时就会产生一个新的文件。对于写操作有两个参数，一个规定了消息的数量达到这个值时必须将数据刷新到硬盘上，另外一个规定了刷新到硬盘的时间间隔，这对数据的持久性是个保证，在系统崩溃的时候只会丢失一定数量的消息或者一个时间段的消息。

**读操作**
读操作需要两个参数：一个64位的offset和一个S字节的最大读取量。S通常比单个消息的大小要大，但在一些个别消息比较大的情况下，S会小于单个消息的大小。这种情况下读操作会不断重试，每次重试都会将读取量加倍，直到读取到一个完整的消息。可以配置单个消息的最大值，这样服务器就会拒绝大小超过这个值的消息。也可以给客户端指定一个尝试读取的最大上限，避免为了读到一个完整的消息而无限次的重试。
在实际执行读取操纵时，首先需要定位数据所在的日志文件，然后根据offset计算出在这个日志中的offset(前面的的offset是整个分区的offset),然后在这个offset的位置进行读取。定位操作是由二分查找法完成的，Kafka在内存中为每个文件维护了offset的范围。
下面是发送给consumer的结果的格式：

```bash
MessageSetSend (fetch result)

total length     : 4 bytes
error code       : 2 bytes
message 1        : x bytes
...
message n        : x bytes
MultiMessageSetSend (multiFetch result)

total length       : 4 bytes
error code         : 2 bytes
messageSetSend 1
...
messageSetSend n
```
**删除**
日志管理器允许定制删除策略;
目前的策略是删除修改时间在N天之前的日志（按时间删除），也可以使用另外一个策略：保留最后的N GB数据的策略(按大小删除)。
为了避免在删除时阻塞读操作，采用了copy-on-write形式的实现，删除操作进行时，读取操作的二分查找功能实际是在一个静态的快照副本上进行的，这类似于Java的CopyOnWriteArrayList。

**可靠性保证**
日志文件有一个可配置的参数M，缓存超过这个数量的消息将被强行刷新到硬盘。
一个日志矫正线程将循环检查最新的日志文件中的消息确认每个消息都是合法的。
合法的标准为：所有文件的大小的和最大的offset小于日志文件的大小，并且消息的CRC32校验码与存储在消息实体中的校验码一致。如果在某个offset发现不合法的消息，从这个offset到下一个合法的offset之间的内容将被移除。
有两种情况必须考虑:
1. 当发生崩溃时有些数据块未能写入
2. 写入了一些空白数据块

inode,但无法保证更新inode和写入数据的顺序;
inode保存的大小信息被更新了，但写入数据时发生了崩溃，就产生了空白数据块。
CRC校验码可以检查这些块并移除，当然因为崩溃而未写入的数据块也就丢失了。