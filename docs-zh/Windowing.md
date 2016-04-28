---
title: Windowing Support in Core Storm
layout: documentation
documentation: true
---

Storm 核心模块已经支持了处理 `window` (一组 tuples 组成)。 窗口可以用以下两个参数指定：
1. Window length -  窗口的长度或时间
2. Sliding interval - 窗口的滑动时间间隔

## 滑动窗口

滑动窗口安装滑动时间间隔滑动，tuples 会被分组到这些滑动窗口中。一个 tuple 可能属于不止一个窗口。
比如一个基于时间的滑动窗口，其窗口长度为 10s，但滑动间隔为 5s。

```
| e1 e2 | e3 e4 e5 e6 | e7 e8 e9 |...
0       5             10         15    -> time

|<------- w1 -------->|
        |------------ w2 ------->|
```

每隔 5s 该窗口滑动一次，其中一个 tuples 即属于第一个窗口也属于第二个窗口。

## 滚动窗口

无论窗口是基于时间的还是基于长度的，任何 tuple 都只会被分组到其中一个窗口中。
如下是一个基于时间的滚动窗口，其长度是 5s：

```
| e1 e2 | e3 e4 e5 e6 | e7 e8 e9 |...
0       5             10         15    -> time
   w1         w2            w3
```

每隔 5s 该窗口滚动一次，窗口没有任何重叠的部分。

Storm 支持指定窗口的长度和滑动间隔（tuples 的数量或时间）。

若使用窗口功能需要实现 `IWindowedBolt` 接口。

```java
public interface IWindowedBolt extends IComponent {
    void prepare(Map stormConf, TopologyContext context, OutputCollector collector);
    /**
     * Process tuples falling within the window and optionally emit
     * new tuples based on the tuples in the input window.
     */
    void execute(TupleWindow inputWindow);
    void cleanup();
}
```

每次窗口启动，就会调用 `execute` 方法。通过 TupleWindow 参数可以访问在窗口中当前 tuples，自从前一窗口计算完后，这些 tuples 就会过期，新的 tuples 会添加到 TupleWindow中，这有助于我们进行有效窗口的计算。

Bolts 如果需要基础的窗口功能，可以继承 `BaseWindowedBolt` 抽象类，其具有指定窗口长度和滑动间隔的 API。

例如：

```java
public class SlidingWindowBolt extends BaseWindowedBolt {
	private OutputCollector collector;

    @Override
    public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
    	this.collector = collector;
    }

    @Override
    public void execute(TupleWindow inputWindow) {
	  for(Tuple tuple: inputWindow.get()) {
	    // do the windowing computation
		...
	  }
	  // emit the results
	  collector.emit(new Values(computedValue));
    }
}

public static void main(String[] args) {
    TopologyBuilder builder = new TopologyBuilder();
     builder.setSpout("spout", new RandomSentenceSpout(), 1);
     builder.setBolt("slidingwindowbolt",
                     new SlidingWindowBolt().withWindow(new Count(30), new Count(10)),
                     1).shuffleGrouping("spout");
    Config conf = new Config();
    conf.setDebug(true);
    conf.setNumWorkers(1);

    StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createTopology());

}
```

支持以下窗口配置：

```java
withWindow(Count windowLength, Count slidingInterval)
基于 Tuple 数量的滑动窗口在 `slidingInterval` 数量的 tuples 后滑动

withWindow(Count windowLength)
基于 Tuple 数量的滑动窗口在每个 tuple 后滑动

withWindow(Count windowLength, Duration slidingInterval)
基于 Tuple 数量的滑动窗口在 `slidingInterval` 时间间隔后滑动

withWindow(Duration windowLength, Duration slidingInterval)
基于时间间隔的滑动窗口在 `slidingInterval` 时间间隔后滑动

withWindow(Duration windowLength)
基于时间间隔的滑动窗口在每个 tuple 后滑动

withWindow(Duration windowLength, Count slidingInterval)
基于时间间隔的滑动窗口在 `slidingInterval` 数量的 tuples 后滑动

withTumblingWindow(BaseWindowedBolt.Count count)
基于 Tuple 数量的滚动窗口在指定数量的 tuples 后滚动

withTumblingWindow(BaseWindowedBolt.Duration duration)
基于时间间隔的滚动窗口在指定时间间隔后滚动
```

## Tuple 时间戳和有序 tuples

窗口中的默认跟踪时间戳是 bolt 处理 tuple 的时间。窗口计算的执行是基于这个时间戳的。Storm 已经支持基于数据源生成的时间戳跟踪窗口。

```java
/**
* 指定 tuple 中的一个字段（long 型）的值来代表时间戳，如果这个字段在输入的 tuple 中不存在，就会抛出  {@link IllegalArgumentException} 异常。
*
* @param fieldName 包含时间戳的字段名
*/
public BaseWindowedBolt withTimestampField(String fieldName)
```

从输入的 tuple 中查找到的上述 `fieldName` 的值会影响窗口的计算。

如何在 tuple 中不存在该字段，将会抛出异常，同时间戳字段名一起，还可以指定一个延迟时间参数，其表示了有时序 tuples 的最大时间限制。

例如：如果延迟是 5s，tuple `t1` 的到达时间是 `06:00:05`，那么将没有时间（携带的时间）早于 `06:00:00` 的 tuple 到达（译注：窗口会忽略掉那些 达到时间 < 当前 tuple 的到达时间 - lag 的 tuples）。如果一个 tuple 在 `t1` 之后到达，但时间是 05:59:59，其对应的窗口已经被移动到 `t1` 之前了。 那么这个 tuple 会被看做是迟到的 tuple 而不被处理。目前，迟到的 tuples 会以 INFO 级别的日志记录到 worker 的日志文件中。

```java
/**
* 指定 tuple 时间（毫秒）的最大延迟，这意味着 tuple 的延迟时间不能大于这个值
*
* @param duration 最大延迟时间
*/
public BaseWindowedBolt withLag(Duration duration)
```

### Watermarks

对于指定时间字段的 tuples
For processing tuples with timestamp field, storm internally computes watermarks based on the incoming tuple timestamp. Watermark is
the minimum of the latest tuple t
imestamps (minus the lag) across all the input streams. At a higher level this is similar to the watermark concept
used by Flink and Google's MillWheel for tracking event based timestamps.

Periodically (default every sec), the watermark timestamps are emitted and this is considered as the clock tick for the window calculation if
tuple based timestamps are in use. The interval at which watermarks are emitted can be changed with the below api.

```java
/**
* Specify the watermark event generation interval. For tuple based timestamps, watermark events
* are used to track the progress of time
*
* @param interval the interval at which watermark events are generated
*/
public BaseWindowedBolt withWatermarkInterval(Duration interval)
```


When a watermark is received, all windows up to that timestamp will be evaluated.

For example, consider tuple timestamp based processing with following window parameters,

`Window length = 20s, sliding interval = 10s, watermark emit frequency = 1s, max lag = 5s`

```
|-----|-----|-----|-----|-----|-----|-----|
0     10    20    30    40    50    60    70
```

Current ts = `09:00:00`

Tuples `e1(6:00:03), e2(6:00:05), e3(6:00:07), e4(6:00:18), e5(6:00:26), e6(6:00:36)` are received between `9:00:00` and `9:00:01`

At time t = `09:00:01`, watermark w1 = `6:00:31` is emitted since no tuples earlier than `6:00:31` can arrive.

Three windows will be evaluated. The first window end ts (06:00:10) is computed by taking the earliest event timestamp (06:00:03)
and computing the ceiling based on the sliding interval (10s).

1. `5:59:50 - 06:00:10` with tuples e1, e2, e3
2. `6:00:00 - 06:00:20` with tuples e1, e2, e3, e4
3. `6:00:10 - 06:00:30` with tuples e4, e5

e6 is not evaluated since watermark timestamp `6:00:31` is older than the tuple ts `6:00:36`.

Tuples `e7(8:00:25), e8(8:00:26), e9(8:00:27), e10(8:00:39)` are received between `9:00:01` and `9:00:02`

At time t = `09:00:02` another watermark w2 = `08:00:34` is emitted since no tuples earlier than `8:00:34` can arrive now.

Three windows will be evaluated,

1. `6:00:20 - 06:00:40` with tuples e5, e6 (from earlier batch)
2. `6:00:30 - 06:00:50` with tuple e6 (from earlier batch)
3. `8:00:10 - 08:00:30` with tuples e7, e8, e9

e10 is not evaluated since the tuple ts `8:00:39` is beyond the watermark time `8:00:34`.

The window calculation considers the time gaps and computes the windows based on the tuple timestamp.

## 消息保障

Storm 核心模块中的窗口功能目前支持至少一次的消息保障。bolts 通过 `execute(TupleWindow inputWindow)` 方法发出的 values 会自动锚于该 TupleWindow 的所有 tuples 上。期待下游 bolts ack 其接受的的 tuple （即：从窗口 bolt 发出的 tuple）来完成 tuple 树。否则这些 tuples 将会被重发，窗口程序也会重新计算。

窗口中的 tuples 过期时（即超过 `windowLength + slidingInterval`）会自动被 acked。注意：对于基于时间的窗口，`topology.message.timeout.secs` 配置项应该远大于 `windowLength + slidingInterval`，否则 tuples 会因为超时而被重新发送，进而导致重复计算；对于基于数量的窗口，这个配置项应该根据 `windowLength + slidingInterval` 内的 tuples 能够在超时期间被接收并处理完而定。

## 示例 topology

示例 topology `SlidingWindowTopology` 展示了如何使用这些 API 来计算滑动窗口的 sum 指标及滚动窗口的平均值指标。
