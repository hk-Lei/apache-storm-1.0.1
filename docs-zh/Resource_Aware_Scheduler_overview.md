---
title: Resource Aware Scheduler
layout: documentation
documentation: true
---

# 简介

这篇文章的目的是提供对分布式计算系统 Storm 的资源感应调度器 (RAS) 的一个介绍。文章将给你提供的是对 Storm 中的资源感应调度器的一个高度抽象的说明。

## 使用资源感应调度器

用户可以通过改变 *conf/storm.yaml* 中的以下配置项使用资源感应调度器

```ymal
    storm.scheduler: "org.apache.storm.scheduler.resource.ResourceAwareScheduler"
```

## API 概览

若使用 Trident，请移步 [Trident RAS API](./Trident-RAS-API.html)

对于一个 Topology，用户可以指定各个组件（如：Spout 或 Bolt）运行时每个实例所需要的资源数。用户可以通过以下 API 指定一个组件的所需资源。

### 设置所需 Memory

设置组件所需内存的 API :

```java
    public T setMemoryLoad(Number onHeap, Number offHeap)
```

参数：
* Number onHeap - 组件的一个实例所使用的堆内存的数量（以 MB 为单位）
* Number offHeap - 组件的一个实例所使用的堆外内存的数量（以 MB 为单位）

如果组件不需要堆外内存，用户也可以选择只指定组件所需的堆内存：

```java
    public T setMemoryLoad(Number onHeap)
```

参数：
* Number onHeap – 该组件的一个实例所使用的堆内存的数量（以 MB 为单位）

如果没有提供堆外内存大小，默认使用堆外内存 0.0MB。如果没有提供堆内存大小或者组件没有调用上述 API，默认值将会被使用：
**译注：** 堆内存和堆外内存的默认值又以下配置项指定
```
topology.component.resources.onheap.memory.mb (default.yaml中指定的是 128MB)
topology.component.resources.offheap.memory.mb (default.yaml中指定的是 0.0MB)
```

示例：

```java
    SpoutDeclarer s1 = builder.setSpout("word", new TestWordSpout(), 10);
    s1.setMemoryLoad(1024.0, 512.0);
    builder.setBolt("exclaim1", new ExclamationBolt(), 3)
                .shuffleGrouping("word").setMemoryLoad(512.0);
```

这个 Topology 所需内存总共 16.5GB，其中，10个 Spout 每个需要 1GB 的堆内存和 0.5GB 的堆外内存，3个 Bolt 每个需要 0.5GB 的堆内存。

### 设置所需 CPU

设置组件所需 CPU 的 API :

```java
    public T setCPULoad(Double amount)
```

参数：
* Number amount – 组件的一个实例所使用的 CPU 数量

目前，一个组件所需要的 CPU 资源数或者一个节点的 CPU 可用资源数都是由一个分数来表示的。CPU 的使用量是一个难以定义的概念，不同的 CPU 架构依据不同的执行任务表现不同，用一个精确的数字表示所有的情况是不可能的。相反，我们约定越过配置方法，主要关心粗粒度的 CPU 使用率，同时仍提供指定更细粒度数量的可能性。

通常情况下，一个物理 CPU 核心为 100 分。你可以根据你的处理器的性能相应的调整这个值。重负载任务可以得到 100 分，那样它就可以使用整个核心；中等负载的任务设置 50 分；轻量级负载设置 25 分；微型任务设置 10 分。在某些情况下，你的一个任务需要生成其他的线程用来帮助处理，这些任务可能需要设置超过 100 分来表达他们对 CPU 的使用。如果遵循这些约定，通常情况下一个单线程任务所需要的 CPU 分值是其容量 * 100。

```java
    SpoutDeclarer s1 = builder.setSpout("word", new TestWordSpout(), 10);
    s1.setCPULoad(15.0);
    builder.setBolt("exclaim1", new ExclamationBolt(), 3)
                .shuffleGrouping("word").setCPULoad(10.0);
    builder.setBolt("exclaim2", new HeavyBolt(), 1)
                    .shuffleGrouping("exclaim1").setCPULoad(450.0);
```

### 限制 Worker 进程 (JVM) 的堆内存大小

```java
    public void setTopologyWorkerMaxHeapSize(Number size)
```
参数：
* Number size – Worker 进程被限制的内存大小(MB)

用户可以使用上述 API，在 Topology 级别限制 RAS 分配给单个 Worker 进程的内存资源。这个 API 可以方便用户可以将 executors 传递到多个进程。但是，传递 executors 到多个进程可能会增加延迟，因为 executors 进程内部通信将不能使用 Disruptor 队列。
**译注：没太明白，求大神指点！**

示例：
```java
    Config conf = new Config();
    conf.setTopologyWorkerMaxHeapSize(512.0);
```

### 设置节点的可用资源

Storm 管理员可以通过修改相应节点上 Storm 安装目录下的 *conf/storm.yaml* 文件指定该节点的可用资源。

Storm 管理员可以在 *conf/storm.yaml* 中添加以下配置项 (单位为 MB) 来指定一个节点的可用内存资源：

```java
    supervisor.memory.capacity.mb: [amount<Double>]
```
Storm 管理员也可在 *conf/storm.yaml* 中添加以下配置项来指定一个节点的可用 CPU 资源：

```java
    supervisor.cpu.capacity: [amount<Double>]
```
**Note：** 用户可以指定的可用 CPU 资源数量是如前所述的使用分数制表示的。
示例：

```yaml
    supervisor.memory.capacity.mb: 20480.0
    supervisor.cpu.capacity: 100.0
```

### 其他配置项

用户可以在 *conf/storm.yaml* 中为 RAS 配置一些默认配置项：

```yaml
    //当堆内存没有被组件指定使的默认值
    topology.component.resources.onheap.memory.mb: 128.0

    //当堆外内存没有被组件指定使的默认值
    topology.component.resources.offheap.memory.mb: 0.0

    //当所需 CPU 资源没有被组件指定时的默认值
    topology.component.cpu.pcore.percent: 10.0

    //当 Worker 进程的最近堆内存没有被指定时的默认值
    topology.worker.max.heap.size.mb: 768.0
```

# Topology 优先级和每个用户资源配置

通常许多 Storm 用户都是共享一个 Storm 集群，因此 RSA 也有多租户的功能。RAS 可以做到在用户级别分配资源。在条件允许的情况下，RAS 可以满足每个用户具有一定数量的资源用于运行自己的 Topology。当 Storm 集群有额外的空闲资源时，RAS 会将其公平的分配给用户。不同的 Topology 的重要性也各不相同，有的 Topology 是用于实际生产中的，有的仅仅是实验性的，因此，RAS 在决定 Topologies 的调度或逐出时的顺序是会考虑 Topologies 的主要性。

## Setup

可以在 *conf/user-resource-pools.yaml* 中指定用户资源的配置项，格式如下：

```yaml
    resource.aware.scheduler.user.pools:
    [UserId]
    cpu: [Amount of Guarantee CPU Resources]
    memory: [Amount of Guarantee Memory Resources]
```

*user-resource-pools.yaml* 示例：

```yaml
    resource.aware.scheduler.user.pools:
        jerry:
            cpu: 1000
            memory: 8192.0
        derek:
            cpu: 10000.0
            memory: 32768
        bobby:
            cpu: 5000.0
            memory: 16384.0
```

请注意，指定 CPU 和 Memory 资源的数值可以是整型也可以是 Double 类型的。

## API 概述
### 指定 Topology 优先级
Topology 的优先级的范围为 0 ~ 29。可以将 Topologies 按照一定的范围分为几大类，例如：

```
    PRODUCTION => 0 – 9
    STAGING => 10 – 19
    DEV => 20 – 29
```

因此，每个分类下包含 10 个子优先级。用户可以通过以下 API 设置 Topology 的优先级：

```java
    conf.setTopologyPriority(int priority)
```

参数：
* priority – 一个表示 Topology 优先级的整数

**注意：** 0 ~ 29 并不是一个硬性限制，因此，用户可以指定大于 29 的数字的优先级。当然，数值越大，优先级越低的规律仍然有效。

### 指定调度策略

用户可以指定 Topology 级别的调度策略。用户可以实现 IStrategy 接口为某些特殊的 Topology 定义新的调度策略。我们意识到不同的 Topology 可能需要不同的调度机制，因此我们抽象了这个插件式的接口。用户可以通过以下 API 在定义 Topology 的时候设置其的调度策略：

```java
    public void setTopologyStrategy(Class<? extends IStrategy> clazz)
```

参数：
* clazz – 实现了 IStrategy 接口的调度策略类

示例：

```java
    conf.setTopologyStrategy(org.apache.storm.scheduler.resource.strategies.schedulin.DefaultResourceAwareStrategy.class);
```

Storm 提供了一个默认的调度策略 - DefaultResourceAwareStrategy，其实现了 Storm 中的 RAS 调度算法，详见[原始论文](http://web.engr.illinois.edu/~bpeng/files/r-storm.pdf)。

### 指定 Topology 优先级策略

调度顺序是一个插件式的接口，因此用户可以定义策略来区分 Topologies 的优先级。若用户需要定义自己的优先级策略，其需要实现 ISchedulingPriorityStrategy 接口。用户可以通过配置 *Config.RESOURCE_AWARE_SCHEDULER_PRIORITY_STRATEGY* 项来设置调度优先级策略，例如：

```yaml
    resource.aware.scheduler.priority.strategy: "org.apache.storm.scheduler.resource.strategies.priority.DefaultSchedulingPriorityStrategy"
```
Storm 提供了一个默认的优先级调度策略，下文会说明这个默认优先级调度策略是如何工作的。

**DefaultSchedulingPriorityStrategy**

调度顺序应该基于用户的目前使用的资源量离分配给他（她）的资源量的差距，我们应该优先考虑差距最大的。这个问题的难点是一个用户可以拥有多个分配资源，另一个用户又有另一套分配资源（个人认为是多个节点资源分配各不相同），我们如何公平的比较他们？ 我们使用平均资源占用百分比的方法来比较他们。
示例：

|用户|保证资源|分配资源|
|----|------------------|------------------|
|A|<10 CPU, 50GB>|<2 CPU, 40 GB>|
|B|< 20 CPU, 25GB>|<15 CPU, 10 GB>|

用户 A 的平均资源占用百分比为：

(2/10+40/50)/2  = 0.5

用户 B 的平均资源占用百分比为：

(15/20+10/25)/2  = 0.575

因此，在这个示例中，用户 A 的平均资源占用百分比比用户 B 小，用户 A 应该优先被分配资源。也就是说，调度用户 A 提交的 Topology。

在调度中，RAS 按照用户的平均资源占用百分比将其排序，然后优先调度占用比最低的 Topology。如果某个用户的分配资源被完全占用了，那么这个用户的平均资源占用百分比将会大于或等于 1。

### 指定逐出策略
逐出策略是用在当集群中已经没有空闲资源再去调度新的 Topologies 的时候。如果集群已满，我们需要一种机制来逐出一些 Topologies，这样可以保证用户的保证分配的资源得到满足已经额外的资源能够公平的共享给用户。逐出 Topologies 的策略也是以接口的方式提供的，这样用户可以实现自己的逐出 Topologies 的策略。如果用户实现自己的逐出策略，需要实现 IEvictionStrategy 接口并且设置 *Config.RESOURCE_AWARE_SCHEDULER_EVICTION_STRATEGY* 配置项的类为自己的实现类，例如：

```yaml
    resource.aware.scheduler.eviction.strategy: "org.apache.storm.scheduler.resource.strategies.eviction.DefaultEvictionStrategy"
```
Storm 提供了一个默认的逐出策略。下文解释了这个默认的逐出策略是如何工作的：

**DefaultEvictionStrategy**

决定 Topology 是否应该被逐出时，我们应该考虑我们调度的这个 Topology 的优先级及这个 Topology 的所属用户的分配资源是否已经被使用完。

我们不应该去逐出一个没满足其用户分配资源的 Topology。下面的流程图描述了逐出过程的逻辑：（**然而并没有图！**）
![Viewing metrics with VisualVM](../docs/images/resource_aware_scheduler_default_eviction_strategy.svg)
