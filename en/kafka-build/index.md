# Kafka集群搭建(伪集群)


[[Kafka]^(分布式事件流处理平台)](https://kafka.apache.org)

本文将介绍kafka伪集群的搭建。

<!--more-->

{{< admonition type=info title="环境说明" open=true >}}

- Ubuntu-64bit-20.04.2

- java-11.0.10

- zookeeper-3.7.0

  {{< link href="https://imicrocode.com/en/zookeeper-build/" content=Zookeeper集群搭建  title="Visit Zookeeper Build Page" >}} 

- kafka-2.7.0

{{< /admonition >}}

## 下载和解压kafka

{{< link href="http://kafka.apache.org/downloads" content=Kafka官方下载  title="Visit Kafka Dwonload Page" >}} 

以路径`/home/micro/kafka`为例

```shell
$ cd /home/micro/kafka
$ wget https://downloads.apache.org/kafka/2.7.0/kafka_2.13-2.7.0.tgz
$ tar -xzvf kafka_2.13-2.7.0.tgz
```

## 创建kafka日志文件夹

```shell
$ mkdir kafka-logs1 kafka-logs2 kafka-logs3
```

## 修改配置文件

复制3个配置文件

```shell
$ cd kafka_2.13-2.7.0/config/
$ cp server.properties server1.properties
$ cp server.properties server2.properties
$ cp server.properties server3.properties
```

修改配置文件中的`broker.id`分别为1、2、3

`listeners` 端口分别为9092、9093、9094

`log.dirs` 分别设置为kafka-logs1、kafka-logs2、kafka-logs3



`server1.properties`的关键配置：

```properties
broker.id=1
listeners=PLAINTEXT://192.168.90.5:9092
log.dirs=/home/micro/kafka/kafka-logs1
# 每个topic的默认partitions数
# 建议配置大于1,否则__consumer_offset分区为1时,broker挂掉无法恢复时会无法消费
num.partitions=3
# 是否能自动创建topic,该参数生产环境建议关闭
auto.create.topics.enable=false
zookeeper.connect=192.168.90.4:2181,192.168.90.4:2182,192.168.90.4:2183
```

`server2.properties`的关键配置：

```properties
broker.id=2
listeners=PLAINTEXT://192.168.90.5:9093
log.dirs=/home/micro/kafka/kafka-logs2
num.partitions=3
auto.create.topics.enable=false
zookeeper.connect=192.168.90.4:2181,192.168.90.4:2182,192.168.90.4:2183
```

`server3.properties`的关键配置：

```properties
broker.id=3
listeners=PLAINTEXT://192.168.90.5:9094
log.dirs=/home/micro/kafka/kafka-logs3
num.partitions=3
auto.create.topics.enable=false
zookeeper.connect=192.168.90.4:2181,192.168.90.4:2182,192.168.90.4:2183
```

## 启动kafka

**启动kafka之前，请确认zk服务已启动**

```shell
$ cd kafka_2.13-2.7.0/bin/
$ ./kafka-server-start.sh -daemon ../config/server1.properties
$ ./kafka-server-start.sh -daemon ../config/server2.properties
$ ./kafka-server-start.sh -daemon ../config/server3.properties
```

## 验证集群是否搭建成功

### 集群下创建Topic

创建一个名为kafka-test的topic

```shell
$ ./kafka-topics.sh --create --topic kafka-test --bootstrap-server 192.168.90.5:9092
```

查看已创建的topic

```shell
$ ./kafka-topics.sh --list kafka-test --bootstrap-server 192.168.90.5:9092
```

删除topic

```shell
$ ./kafka-topics.sh --delete --topic kafka-test --bootstrap-server 192.168.90.5:9092
```



### 集群下启动Consumer

在一个新的终端中:

```shell
$ ./kafka-console-consumer.sh  --bootstrap-server 192.168.90.5:9092,192.168.90.5:9093,192.168.90.5:9094 --topic kafka-test
```

### 集群下启动Producer

在一个新的终端中：

```shell
$ ./kafka-console-producer.sh --bootstrap-server 192.168.90.5:9092,192.168.90.5:9093,192.168.90.5:9094 --topic kafka-test
```

### 集群下Producer终端下发送消息

在生产者窗口输入任意数据，观察Consumer窗口的变化

## 结语

以上是搭建kafka伪集群的全过程，真正集群的搭建也均类似。
