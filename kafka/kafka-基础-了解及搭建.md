## 一. kafka结构
特点：
1. 支持高Throughput的应用
2. scale out：无需停机即可扩展机器
3. 持久化：通过将数据持久化到硬盘以及replication防止数据丢失
4. 支持online和offline的场景。

消息层次：
1. Topic：一类消息，例如page view日志，click日志等都可以以topic的形式存在，kafka集群能够同时负责多个topic的分发
2. Partition： Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。
3. Message：消息，最小订阅单元
！[](http://img.blog.csdn.net/20131216165124968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWZraXNz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
Producer：消息发布者
Broker：消息中间件处理结点，一个kafka节点就是一个broker
Consumer：消息订阅者

具体流程：
1. Producer根据指定的partition方法（round-robin、hash等），将消息发布到指定topic的partition里面
2. kafka集群接收到Producer发过来的消息后，将其持久化到硬盘，并保留消息指定时长（可配置），而不关注消息是否被消费。
3. Consumer从kafka集群pull数据，并控制获取消息的offset

## 二. kafka设计考虑
**吞吐率**
High Throughput是kafka需要实现的核心目标之一，为此kafka做了以下一些设计：
1. 数据磁盘持久化：消息不在内存中cache，直接写入到磁盘，充分利用磁盘的顺序读写性能
2. zero-copy：减少IO操作步骤
3. 数据批量发送
4. 数据压缩
5. Topic划分为多个partition，提高并行性

**负载均衡/高可用**
1. producer根据用户指定的算法，将消息发送到指定的partition
2. 存在多个partiiton，每个partition有自己的replica，每个replica分布在不同的Broker节点上
3. 多个partition需要选取出lead partition，lead partition负责读写，并由zookeeper负责fail over
4. 通过zookeeper管理broker与consumer的动态加入与离开

**适合pull数据**
由于kafka broker会持久化数据，broker没有内存压力，因此，consumer非常适合采取pull的方式消费数据，具有以下几点好处：
1. 简化kafka设计
2. consumer根据消费能力自主控制消息拉取速度
3. consumer根据自身情况自主选择消费模式，例如批量，重复消费，从尾端开始消费等

**横向扩展**
当需要增加broker结点时，新增的broker会向zookeeper注册，而producer及consumer会根据注册在zookeeper上的watcher感知这些变化，并及时作出调整。

## 三. 环境搭建
启动zookeeper
准备启动一个3个broker节点的kafka集群

```bash
# server0.properties
config/server0.properties:
    broker.id=0
    port=9092
    log.dirs=/home/zsh/tmp/kafka/kafka-logs-0

# server1.properties
config/server0.properties:
    broker.id=1
    port=9093
    log.dirs=/home/zsh/tmp/kafka/kafka-logs-1

# server2.properties
config/server0.properties:
    broker.id=2
    port=9094
    log.dirs=/home/zsh/tmp/kafka/kafka-logs-2
```
参数：
- broker.id: broker节点的唯一标识
- port: broker节点使用端口号
- logs.dir: 消息目录位置

启动三个broker

```bash
bin/kafka-server-start.sh config/server0.properties &
bin/kafka-server-start.sh config/server1.properties &
bin/kafka-server-start.sh config/server2.properties &
```

## 四. 本地测试
**创建topic**

```bash
#创建一个名为test的topic，只有一个副本，一个分区
kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```
**查看topic**

```bash
kafka-topics.sh -list -zookeeper localhost:2181
```
**启动producer**

```bash
# 启动producer连接某个端口9092，然后发送消息
kafka-console-producer.sh --broker-list localhost:9092 --topic test
test
hello boy
```
**启动consumer**

```bash
#启动consumer之后就可以在console中看到producer发送的消息了
kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
```

## 五. java开发
kafka服务端，不能配置为localhost，否则会连接不上；
配置参数

```java
public interface KafkaProperties {
    final static String ubuntu_ip = "192.168.188.129";
    final static String zkConnect = ubuntu_ip+":2181";
    final static String groupId = "group1";
    final static String topic = "zsh-topic";
    final static String kafkaServerURL = ubuntu_ip;
    final static int kafkaServerPort = 9092;
    final static int kafkaProducerBufferSize = 64 * 1024;
    final static int connectionTimeOut = 20000;
    final static int reconnectInterval = 10000;
    final static String topic2 = "topic2";
    final static String topic3 = "topic3";
    final static String clientId = "SimpleConsumerDemoClient";
}
```
producer

```java
public class KafkaProducer extends Thread {
    private final kafka.javaapi.producer.Producer<Integer, String> producer;
    private final String topic;
    private final Properties props = new Properties();
    public KafkaProducer(String topic) {
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        props.put("metadata.broker.list", KafkaProperties.ubuntu_ip+":9092");
        producer = new kafka.javaapi.producer.Producer<Integer, String>(new ProducerConfig(props));
        this.topic = topic;
    }
    @Override
    public void run() {
        int messageNo = 1;
        while (true)
        {
            String messageStr = new String("Message_" + messageNo);
            System.out.println("Send:" + messageStr);
            producer.send(new KeyedMessage<Integer, String>(topic, messageStr));
            messageNo++;
            try {
                sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        KafkaProducer producerThread = new KafkaProducer(KafkaProperties.topic);
        producerThread.start();
    }
}
```
consumer

```java
public class KafkaConsumer extends Thread {
    private final ConsumerConnector consumer;
    private final String topic;
    public KafkaConsumer(String topic) {
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(
                createConsumerConfig());
        this.topic = topic;
    }
    private static ConsumerConfig createConsumerConfig() {
        Properties props = new Properties();
        props.put("zookeeper.connect", KafkaProperties.zkConnect);
        props.put("group.id", KafkaProperties.groupId);
        props.put("zookeeper.session.timeout.ms", "40000");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");
        return new ConsumerConfig(props);
    }
    @Override
    public void run() {
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, new Integer(1));
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        KafkaStream<byte[], byte[]> stream = consumerMap.get(topic).get(0);
        ConsumerIterator<byte[], byte[]> it = stream.iterator();
        while (it.hasNext()) {
            System.out.println("receive：" + new String(it.next().message()));
            try {
                sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        KafkaConsumer consumerThread = new KafkaConsumer(KafkaProperties.topic);
        consumerThread.start();
    }
}
```