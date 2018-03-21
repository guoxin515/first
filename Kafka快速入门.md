# Kafka快速入门 #

**1、环境的安装和启动**

下载安装包：Download the 1.0.1 release and un-tar it.
https://www.apache.org/dyn/closer.cgi?path=/kafka/1.0.1/kafka_2.11-1.0.1.tgz

解压压缩包	

> tar -xzf kafka_2.11-1.0.1.tgz

启动zookeeper	
> bin/zookeeper-server-start.sh config/zookeeper.properties

启动kafka	
> bin/kafka-server-start.sh config/server.properties

创建topic	
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

查看已有topic
> bin/kafka-topics.sh --list --zookeeper localhost:2181

Alternatively, instead of manually creating topics you can also configure your brokers to auto-create topics when a non-existent topic is published to（或者，您也可以将代理配置为在发布不存在的主题时自动创建主题，而不是手动创建主题）

生产者（Producer）发送消息 
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

消费者（Consumer ）发送消息
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

**2、集群配置**
创建多个kafka配置
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties

修改配置文件：
> config/server-1.properties:
>     broker.id=1
>     listeners=PLAINTEXT://:9093
>     log.dir=/tmp/kafka-logs-1
>  
> config/server-2.properties:
>     broker.id=2
>     listeners=PLAINTEXT://:9094
>     log.dir=/tmp/kafka-logs-2

启动另外的两个kafka集群：
We already have Zookeeper and our single node started, so we just need to start the two new nodes:
> bin/kafka-server-start.sh config/server-1.properties &
...
> bin/kafka-server-start.sh config/server-2.properties &
...

创建一个topic,为3副节点：
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic

查看运行topic的描述信息：
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
> Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
>     Topic: my-replicated-topic  Partition: 0    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0

查看原来创建的topic:
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test  PartitionCount:1    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0

新topic,发送消息
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C

消费消息：
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C

Now let's test out fault-tolerance. Broker 1 was acting as the leader so let's kill it:
现在我们来测试容错。服务1充当领导者，所以让我们杀了它：
> ps aux | grep server-1.properties
7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.8/Home/bin/java...
> kill -9 7564

On Windows use:
> wmic process where "caption = 'java.exe' and commandline like '%server-1.properties%'" get processid
ProcessId
6016
> taskkill /pid 6016 /f

Leadership has switched to one of the slaves and node 1 is no longer in the in-sync replica set:
领导层已切换到其中一个从属节点，并且节点1不再处于同步副本集中
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 2   Replicas: 1,2,0 Isr: 2,0

消费者仍然可以消费：
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C

Kafka Connect是Kafka附带的一个工具，可以将数据导入和导出到Kafka。它是一个可扩展的工具，运行连接器，实现与外部系统交互的自定义​​逻辑。在这个快速入门中，我们将看到如何使用简单的连接器运行Kafka Connect，这些连接器将数据从文件导入到Kafka主题，并将数据从Kafka主题导出到文件。

> echo -e "foo\nbar" test.txt

> bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties


> more test.sink.txt
foo
bar



> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
...


> echo Another line>> test.txt