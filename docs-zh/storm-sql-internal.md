---
title: The Internals of Storm SQL
layout: documentation
documentation: true
---

这篇文章描述了 Storm SQL 的设计和实现。

## 概览

SQL 是一种友好的但是却很复杂的标准。很多项目包括 Drill、Hive、Phoenix 及 Spark 都在其 SQL 层面投入很大。StormSQL 的主要设计目标就是利用现有的资源。StormSQL 利用 [Apache Calcite](///calcite.apache.org) 来实现 SQL 标准，而 StormSQL 则主要集中在编译 SQL 语句到 Storm/Trident topologies，这样他们就可以运行在 Storm 集群之上。

图 1 描述了在 StormSQL 中执行一个 SQL 查询的过程。首先，用户提供一系列 SQL 语句，StormSQL 解析 SQL 语句并将其转换成 Calcite 的逻辑计划。由一系列 SQL 逻辑操作符组成的一个逻辑计划，其描述的查询执行过程应该与底层的执行引擎无关。逻辑操作符例如 `TableScan`, `Filter`, `Projection` 及 `GroupBy` 等.

<div align="center">
<img title="Workflow of StormSQL" src="images/storm-sql-internal-workflow.png" style="max-width: 80rem"/>

<p>Figure 1: Workflow of StormSQL.</p>
</div>

The next step is to compile the logical execution plan down to a physical execution plan. A physical plan consists of physical operators that describes how to execute the SQL query in *StormSQL*. Physical operators such as `Filter`, `Projection`, and `GroupBy` are directly mapped to operations in Trident topologies. StormSQL also compiles expressions in the SQL statements into Java byte codes and plugs them into the Trident topologies.

Finally, StormSQL packages both the Java byte codes and the topology into a JAR and submits it to the Storm cluster. Storm schedules and executes the JAR in the same way of it executes other Storm topologies.

The follow code blocks show an example query that filters and projects results from a Kafka stream.

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' ...

CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' ...

INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

The first two SQL statements define the inputs and outputs of external data. Figure 2 describes the processes of how StormSQL takes the last `SELECT` query and compiles it down to Trident topology.

<div align="center">
<img title="Compiling the example query to Trident topology" src="images/storm-sql-internal-example.png" style="max-width: 80rem"/>

<p>Figure 2: Compiling the example query to Trident topology.</p>
</div>


## Constraints of querying streaming tables

There are several constraints when querying tables that represent a real-time data stream:

* The `ORDER BY` clause cannot be applied to a stream.
* There is at least one monotonic field in the `GROUP BY` clauses to allow StormSQL bounds the size of the buffer.

For more information please refer to http://calcite.apache.org/docs/stream.html.

## Dependency

StormSQL does not ship the dependency of the external data sources in the packaged JAR. The users have to provide the dependency in the `extlib` directory of the worker node.
