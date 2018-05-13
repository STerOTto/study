# kafka安装
### Java环境安装
java 1.8
### kafka下载
<http://kafka.apache.org/downloads.html>
### 解压安装包
```bash
$ tar -xzf kafka_2.11-1.1.0.tgz
$ /Users/sterotto/Programs/kafka_2.12-1.1.0
```
### 启动kafka
kafka依赖zookeeper，所以需要启动一个zookeeper服务。kafka中已经内嵌了一个zookeeper，单机启动时，无需额外安装zookeeper。
* zookeeper配置
```bash
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
# disable the per-ip limit on the number of connections since this is a non-production config
maxClientCnxns=0
```
* 启动zookeeper
```Bash
$ bin/zookeeper-server-start.sh config/zookeeper.properties
$ [2018-05-05 10:37:43,638] INFO Starting server (org.apache.zookeeper.server.ZooKeeperServerMain)
```
* kafka配置
```Bash
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=0
# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400
```
* 启动kafka
```bash
$ bin/kafka-server-start.sh config/server.properties
[2018-05-05 11:01:09,783] INFO Kafka version : 1.1.0 (org.apache.kafka.common.utils.AppInfoParser)
[2018-05-05 11:01:09,783] INFO Kafka commitId : fdcf75ea326b8e07 (org.apache.kafka.common.utils.AppInfoParser)
[2018-05-05 11:01:09,784] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
```
* 创建一个topic
```bash
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic com.sterotto.test
$ bin/kafka-topics.sh --list --zookeeper localhost:2181
com.sterotto.test
```
* 发送消息<br/>
    启动一个生产者，通过控制台发送消息
```bash
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic com.sterotto.test
> hello, any body?
> why history message can be received
```
* 启动一个消费组<br/>
```bash
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic com.sterotto.test --from-beginning
hello, any body?
why history message can be received
```
### 启动一个kafka集群
- **1. 为每个broker复制一个配置文件**
```bash
$ cp config/server.properties config/server-1.properties
$ cp config/server.properties config/server-2.properties
```
- **2. 修改配置**
```bash
config/server-1.properties:
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dir=/tmp/kafka-logs-1
 
config/server-2.properties:
    broker.id=2
    listeners=PLAINTEXT://:9094
    log.dir=/tmp/kafka-logs-2
```
其中```broker.id```是每个节点在集群中的唯一和永久的名字，```listeners```是broker监听的端口号，```log.dir```是*kafka*日志存储的位置。
- **3. 启动新增的两个节点**
```bash
$ bin/kafka-server-start.sh config/server-1.properties &
$ bin/kafka-server-start.sh config/server-2.properties &
```
- **4. 创建一个副本数为3的topic**
```bash
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic com.sterotto.replicated.topic --replication-factor 3 --partitions 1
$ bin/kafka-topics.sh --list --zookeeper localhost:2181
com.sterotto.replicated.topic
$ bin/kafka-topics.sh --describe -zookeeper localhost:2181 --topic com.sterotto.replicated.topic
Topic: com.sterotto.replicated.topic	PartitionCount:1	ReplicationFactor:3	Configs:
Topic: com.sterotto.replicated.topic	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```
```describe```命令输出第一行显示的是所有partitions的摘要信息，接下来每一行是每个partition的详细信息，因此可以看到这个topic只有一个partition。
> 1. ```Leader```负责该partition的所有读和写，每个节点随机选取一部分partition作为其leader。（为了保证较高的处理效率，消息的读写都是在固定的一个副本上完成。这个副本所在节点就是所谓的Leader，而其他副本所在节点则是Follower。而Follower则会定期地同步Leader上的数据。）
> 2. ```Replicas```是一个备份了该partition的节点的列表，不论他们是否是leader，也不管这些节点目前是否活着。
> 3. ```Isr```是一个处于同步的副本节点集合。这是replicas的活着的并能与leader保持同步的节点的子集。（如果某个分区所在的leader服务器出了问题，不可用，kafka会从该分区的其他的副本中选择一个作为新的Leader。之后所有的读写就会转移到这个新的Leader上。现在的问题是应当选择哪个作为新的Leader。显然，只有那些跟Leader保持同步的Follower才应该被选作新的Leader。所以会从isr中选取新的Leader，通过ISR，kafka需要的冗余度较低，可以容忍的失败数比较高。假设某个topic有f+1个副本，kafka可以容忍f个服务器不可用。）
- **5. 生产和消费消息，9092端口的broker为```Leader```** 

启动生产者
    
```bash
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic com.sterotto.replicated
>
> hello, world
```
启动消费者

```bash
$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic com.sterotto.replicated
hello, world

```
容错测试，broker0为Leader节点，现在kill它
```Bash
$ ps aux | grep server.properties
wangbo            1108   0.1  2.5  7359484 415900 s001  S+    9:20上午   1:23.56 /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/bin/java
$ kill -9 1108
$ bin/kafka-topics.sh --describe -zookeeper localhost:2181 --topic com.sterotto.replicated.topic
Topic: com.sterotto.replicated.topic	PartitionCount:1	ReplicationFactor:3	Configs:
Topic: com.sterotto.replicated.topic	Partition: 0	Leader: 1	Replicas: 0,1,2	Isr: 1,2
```
现在Leader变成了broker1，而且broker0不再是同步节点

- **6. 使用Kafka Connect 导入/导出数据**


使用场景：使用其他系统的数据源或者使用kafka导出数据到其他系统。Kafka Connect是一个使用kafka导入/导出数据到kafka的工具。这是可扩展的工具，可以实现与其他系统的交互。在本文中，可以看到如何使用kafka连接器进行简单的导入导出数据－－从kafka到文件

第一步创建一个文件
```Bash
echo -e "foo\nbar" > ~/test.txt
```
第二步使用```connect-standalone.sh```启动两个独立的连接器，第一个参数是kafka连接处理器的配置，包含一些通常的配置，例如kafka brokers以及数据序列化格式等。其他的配置每个都指定了需要创建的连接器。这些文件包含一个独一无二的连接器名字，连接器类别，以及连接器需要的其他配置。
```Bash
$ bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties
```

这些简单的配置文件都包含在kafka中，配置文件使用默认本地集群配置，创建两个连接器：第一个是源连接器，从输入文件中按行读取消息，然后发送消息到kafka topic中；第二个是目的连接器，从kafka topic读取消息，然后按行输出到文件中。
第三步启动一个消费者进行消费
```Bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
```
一旦kafka连接器进程已经启动，源连接器应当从test.txt中读取消息，然后将它们发往topic connect－test，同时目的连接器应当开始从connect－test中读取消息，然后将它们写入test.sink.txt中。我们可以确认一下通过数据管道传递过来的数据是否和发送的数据一致。
```Bash
$ cat test.sink.txt
foo
bar
```

参考：http://www.infoq.com/cn/profile/%E9%83%AD%E4%BF%8A



