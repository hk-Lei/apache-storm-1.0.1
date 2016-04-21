---
title: Storm SQL integration
layout: documentation
documentation: true
---

Storm SQL 集成允许用户在 Storm 流式数据中运行 SQL 查询。在流式分析中，SQL 接口不仅会加快开发周期，而且开辟了统一批处理 [Apache Hive](///hive.apache.org) 和实时流式数据处理的机会。

StormSQL 会将 SQL 查询高水准的编译为 [Trident](Trident-API-Overview.html) topologies 并且在 Storm 集群上允许他们。这篇文章将给用户介绍如何使用 StormSQL。如果有人对 StormSQL 的设计和实现的细节感兴趣，请参考 [这里](storm-sql-internal.html)

## 使用

允许 ``storm sql`` 命令编译 SQL 语句为 Trident topology，并且提交到 Storm 集群。

```bash
$ bin/storm sql <sql-file> <topo-name>
```

这里 `sql-file` 包含需要执行的 SQL 语句，`topo-name` 是提交的 topology 的名字。

## 支持的功能

在目前的报表库(1.0.0)中，支持以下功能：

* 流处理读取及写入外部数据源
* 过滤 tuples
* 预测（Projections）

## 指定外部数据源

StormSQL 数据是由外部表的形式表现的，用户可以使用 `CREATE EXTERNAL TABLE` 语句指定数据源。`CREATE EXTERNAL TABLE` 的语法严格遵循 [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL) 中的定义。

```
CREATE EXTERNAL TABLE table_name field_list
    [ STORED AS
      INPUTFORMAT input_format_classname
      OUTPUTFORMAT output_format_classname
    ]
    LOCATION location
    [ TBLPROPERTIES tbl_properties ]
    [ AS select_stmt ]
```

你可以在 [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL) 中找到各属性的详细解释。例如：下列语句指定了一个 Kafka spout 和 sink：

```
CREATE EXTERNAL TABLE FOO (ID INT PRIMARY KEY) LOCATION 'kafka://localhost:2181/brokers?topic=test' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
```

## 对接外部数据源

用户对接外部数据源需要实现 `ISqlTridentDataSource` 接口并且使用 Java 的服务加载机制注册他们，外部数据源就会基于表中 URI 的 Scheme 来选择。请参阅 `storm-sql-kafka` 来了解更多实现细节。

## 示例: 过滤 Kafka 数据流

假设有一个 Kafka 数据流存储交易的订单数据。流中的每个消息包含订单的 id 、产品的单价及订单的数量。我们的目的是过滤出有很大交易额的订单，将这些订单插入另一个 Kafka 数据流用于进行进一步分析。

用户可以在 SQL 文件中指定如下的 SQL 语句：

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

第一条语句定义的 `ORDER` 表代表了输入流。 `LOCATION` 字段指定了 ZK 地址 (`localhost:2181`) 、brokers 在 Zookeeper 中的路径 (`/brokers`) 以及 topic (`orders`)。`TBLPROPERTIES` 字段指定了 [KafkaProducer](http://kafka.apache.org/documentation.html#producerconfigs) 的配置项。
目前 `storm-sql-kafka`的实现即使 table 是 read-only 或 write-only 情况都需要指定 `LOCATION` 和 `TBLPROPERTIES` 项。

类似的第二条语句定义的 `LARGE_ORDERS` 表代表了输出流。第三条 `SELECT` 语句定义了 topology : 其使 StormSQL 过滤外部表 `ORDERS` 中的所有订单(译注：过滤出总价在 50 以上的订单)，计算总价格并将满足的记录插入指定的 `LARGE_ORDER` Kafka 流中。

要运行这个示例，用户需要在 classpath 中包含数据源 (这个示例中是 `storm-sql-kafka`) 及其依赖。一种办法是将所需的 jars 放到 `extlib` 目录中：

```bash
$ cp curator-client-2.5.0.jar curator-framework-2.5.0.jar zookeeper-3.4.6.jar
 extlib/
$ cp scala-library-2.10.4.jar kafka-clients-0.8.2.1.jar kafka_2.10-0.8.2.1.jar metrics-core-2.2.0.jar extlib/
$ cp json-simple-1.1.1.jar extlib/
$ cp jackson-annotations-2.6.0.jar extlib/
$ cp storm-kafka-*.jar storm-sql-kafka-*.jar storm-sql-runtime-*.jar extlib/
```

接下来向 StormSQL 提交 SQL 语句：

```bash
$ bin/storm sql order_filtering order_filtering.sql
```

现在你应该能够在 Storm UI 中看到 `order_filtering` topology。

## 当前缺陷

聚合(Aggregation)、 窗口(windowing)和连表(joining) 尚未实现；暂不支持指定 topology 的并行度；所有处理任务的并行度都是 1。

用户还需要在 `extlib` 目录中提供外部数据源的依赖，否则 topology 将因为 `ClassNotFoundException` 而无法运行。

StormSQL 中当前 Kafka 实现连接器假定输入和输出数据都是JSON格式。连接器还不支持 `INPUTFORMAT` 和 `OUTPUTFORMAT`。
