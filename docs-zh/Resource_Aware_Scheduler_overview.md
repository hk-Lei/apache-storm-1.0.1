---
title: Resource Aware Scheduler
layout: documentation
documentation: true
---
# 说明

这篇文章的目的是提供对分布式计算系统 Storm 的资源感应调度器 (RAS) 的一个介绍。文章将给你提供的是对 Storm 中的资源感应调度器的一个高度抽象的说明。

## 使用资源感应调度器

用户可以通过改变 *conf/storm.yaml* 中的以下配置项使用资源感应调度器

    storm.scheduler: “org.apache.storm.scheduler.resource.ResourceAwareScheduler”

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
topology.component.resources.onheap.memory.mb (default.yaml中指定的是 128MB)
topology.component.resources.offheap.memory.mb (default.yaml中指定的是 0.0MB)

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

目前，一个组件所需要的 CPU 资源数或者一个节点的 CPU 可用资源数都是由一个分数来表示的。CPU 的使用量是一个难以定义的概念，不同的 CPU 架构依据不同的执行任务表现不同，用一个精确的数字表示所有的情况是不可能的。相反，我们约定优于配置的方法，主要是关心粗粒度的 CPU 使用率，同时仍提供指定数量更细粒度的可能性（**求翻译**）。

通常情况下，一个物理 CPU 核心为 100 分。你可以根据你的处理器的性能相应的调整这个值。重负载任务可以得到 100 分，那样它就可以使用整个核心；中等负载的任务设置 50 分；轻量级负载设置 25 分；微型任务设置 10 分。在某些情况下，你的一个任务需要生成其他的线程用来帮助处理，这些任务可能需要设置超过 100 分来表达他们对 CPU 的使用。如果遵循这些约定，通常情况下一个单线程任务所需要的 CPU 分值是其容量 * 100。

```java
    SpoutDeclarer s1 = builder.setSpout("word", new TestWordSpout(), 10);
    s1.setCPULoad(15.0);
    builder.setBolt("exclaim1", new ExclamationBolt(), 3)
                .shuffleGrouping("word").setCPULoad(10.0);
    builder.setBolt("exclaim2", new HeavyBolt(), 1)
                    .shuffleGrouping("exclaim1").setCPULoad(450.0);
```

### 限制 Worker 进程(JVM) 的堆内存大小

```java
    public void setTopologyWorkerMaxHeapSize(Number size)
```
参数：
* Number size – Worker 进程被限制的内存大小(MB)

用户可以使用上述 API 在每个 Topology 级别限制 RAS 分配给单个 Worker 进程的内存资源大小，这个 API 是内置的，所以 
示例：
```java
    Config conf = new Config();
    conf.setTopologyWorkerMaxHeapSize(512.0);
```

### 设置节点的可用资源

```java
    supervisor.memory.capacity.mb: [amount<Double>]
```

```java
    supervisor.cpu.capacity: [amount<Double>]
```

示例：
```yaml
    supervisor.memory.capacity.mb: 20480.0
    supervisor.cpu.capacity: 100.0
```

### 其他配置项

```yaml
    //default value if on heap memory requirement is not specified for a component 
    topology.component.resources.onheap.memory.mb: 128.0

    //default value if off heap memory requirement is not specified for a component 
    topology.component.resources.offheap.memory.mb: 0.0

    //default value if CPU requirement is not specified for a component 
    topology.component.cpu.pcore.percent: 10.0

    //default value for the max heap size for a worker  
    topology.worker.max.heap.size.mb: 768.0
```

# Topology 优先级和每个用户资源配置



## Setup

```yaml
    resource.aware.scheduler.user.pools:
	[UserId]
		cpu: [Amount of Guarantee CPU Resources]
		memory: [Amount of Guarantee Memory Resources]
```

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

## API 概述
### 指定 Topology 优先级

```
    PRODUCTION => 0 – 9
    STAGING => 10 – 19
    DEV => 20 – 29
```

```java
    conf.setTopologyPriority(int priority)
```

### 指定调度策略

```java
    public void setTopologyStrategy(Class<? extends IStrategy> clazz)
```

```java
    conf.setTopologyStrategy(org.apache.storm.scheduler.resource.strategies.schedulin.DefaultResourceAwareStrategy.class);
```

http://web.engr.illinois.edu/~bpeng/files/r-storm.pdf

### 指定 Topology 优先级策略

```yaml
    resource.aware.scheduler.priority.strategy: "org.apache.storm.scheduler.resource.strategies.priority.DefaultSchedulingPriorityStrategy"
```

**DefaultSchedulingPriorityStrategy**


示例：

|User|Resource Guarantee|Resource Allocated|
|----|------------------|------------------|
|A|<10 CPU, 50GB>|<2 CPU, 40 GB>|
|B|< 20 CPU, 25GB>|<15 CPU, 10 GB>|

User A’s average percentage satisfied of resource guarantee: 

(2/10+40/50)/2  = 0.5

User B’s average percentage satisfied of resource guarantee: 

(15/20+10/25)/2  = 0.575

### 指定逐出策略

```yaml
    resource.aware.scheduler.eviction.strategy: "org.apache.storm.scheduler.resource.strategies.eviction.DefaultEvictionStrategy"
```


**DefaultEvictionStrategy**


![Viewing metrics with VisualVM](../docs/images/resource_aware_scheduler_default_eviction_strategy.svg)