---
title: Trident RAS API
layout: documentation
documentation: true
---

## Trident RAS API

Trident RAS(资源感知调度器) API 提供了用户指定 Trident Topology 组件资源的方法。这个 API 看起来确实和基础的 RAS API 一样，只是其实被 Trident 流调用而不是被 Bolts 或 Spouts。

为了避免文档变化导致的不一致情况，资源设置的目的及影响就不在这赘述，如有需要请参见[Resource Aware Scheduler Overview](Resource_Aware_Scheduler_overview.html)

### Use

首先，示例如下：

```java
    TridentTopology topology = new TridentTopology();
    TridentState wordCounts =
        topology
            .newStream("words", feeder)
            .parallelismHint(5)
            .setCPULoad(20)
            .setMemoryLoad(512,256)
            .each( new Fields("sentence"),  new Split(), new Fields("word"))
            .setCPULoad(10)
            .setMemoryLoad(512)
            .each(new Fields("word"), new BangAdder(), new Fields("word!"))
            .parallelismHint(10)
            .setCPULoad(50)
            .setMemoryLoad(1024)
            .groupBy(new Fields("word!"))
            .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))
            .setCPULoad(100)
            .setMemoryLoad(2048);
```

可以对每个操作（Operation）设置资源（除了 grouping、shuffling、partitioning）。Trident 若将一些 Operations 合并为一个 Bolt，那么，这个 Bolt 将拥有这些 Operations 的资源总和。

无论用户设置如何，都会给予每个 Bolt **at least** 的默认资源。

在上述示例中，我们最终得到：

- 一个 Spout 和一个 Spout Coordinator 分别拥有 20% 的 CPU 负载以及 512M 的堆内存和 256 的堆外内存。
- 一个 Bolt 拥有 `Split` 和 `BangAdder` 的资源之和：60% 的 CPU 负载以及 1536M(1024 + 512) 的堆内存
- 一个 Bolt 拥有 100% 的 CPU 负载和 2048M 的堆内存以及默认值的堆外内存资源。

这个 API 可被多次调用。
它有可能在每个 Operation 之后调用；或一些 Operations 之后调用；或者以同样的方式在 `parallelismHint()` 之后调用，用来设置某个整体的资源。 
声明资源同设置并行度有一样的 *boundaries* 。他们不能越过如何的 groupings、shufflings 或者其他重新分区的操作。
