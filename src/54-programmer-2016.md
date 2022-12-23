## Spark Streaming 与 Kafka 集成分析

文/祝海林

>Spark Streaming 诞生于2013年，是 Spark 平台上的流式处理解决方案。本文主要介绍 Spark Streaming 数据接收流程模块中与 Kafka 集成相关的功能。

Spark Streaming 与 Kafka 集成接受数据的方式有两种：

- Receiver-based Approach

- Direct Approach (No Receivers)

我们会对这两种方案做详细解析，同时对比两种方案优劣。选型后，我们针对 Direct Approach（No Receivers）模式详细介绍其如何实现 Exactly Once Semantics，也就是保证接收到的数据只被处理一次，不丢，不重。

### 基于接收器的方式（Receiver-based Approach）

要描述清楚 Receiver-based Approach，我们需要了解其接收流程，分析其内存使用以及相关参数配置对内存的影响。

#### 数据接收流程 

启动 Spark Streaming （后续缩写为 SS）后，SS 会选择一台 Executor 启动 ReceiverSupervisor，并且标记为 Active 状态。接着按如下步骤处理：

- ReceiverSupervisor 会启动对应的 Receiver （这里是 KafkaReceiver）；

- KafkaReceiver 会根据配置启动新的线程接受数据，在该线程中调用 ReceiverSupervisor.store 方法填充数据，注意，这里是一条一条填充的；

- ReceiverSupervisor 会调用 BlockGenerator.addData 进行数据填充。

到目前为止，整个过程不会有太多内存消耗，正常的一个线性调用。所有复杂的数据结构都隐含在 BlockGenerator 中。

#### BlockGenerator 存储结构

BlockGenerator 会复杂些，重要的数据存储结构有四个：

- 维护了一个缓存 currentBuffer，就是一个无限长度的 ArrayBuffer。currentBuffer 并不会被复用，而是每次都会新建，然后把老的对象直接封装成 Block，BlockGenerator 会负责保证 currentBuffer 只有一个。currentBuffer 填充的速度是可以被限制的，以秒为单位，配置参数为 spark.streaming.receiver.maxRate。填充 currentBuffer 是阻塞的，消费 Kafka 的线程直接做填充。

- 维护了一个 blocksForPushing 队列，size 默认为10个（1.5.1版），可通过 spark.streaming.blockQueueSize 进行配置。该队列主要用来实现生产-消费模式。每个元素其实是一个 currentBuffer 形成的 block。

- blockIntervalTimer 是一个定时器。其实是一个生产者，负责将 currentBuffer 的数据放到 blocksForPushing 中。通过参数 spark.streaming.blockInterval 设置，默认为200 ms。放的方式很简单，直接把 currentBuffer 做为 Block 的数据源。这就是为什么 currentBuffer 不会被复用。

- blockPushingThread 也是一个定时器，负责将 Block 从 blocksForPushing 取出来，然后交给 BlockManagerBasedBlockHandler.storeBlock 方法。10 ms 会取一次，不可配置。到这一步，才真的将数据放到了 Spark 的 BlockManager 中。

下面我们会详细分析每一个存储对象对内存的使用情况。

#### currentBuffer

首先自然要说下 currentBuffer，如果200 ms 期间你从 Kafka 接受的数据足够大，则足以把内存承包了。而且 currentBuffer 使用的并不是 Spark 的 Storage 内存，而是有限的用于运算存储的内存。默认应该是 heap*0.4。除了把内存爆掉，还有一个是 GC。导致 Receiver 所在的 Executor 极容易挂掉，处理速度也巨慢。如果你在  SparkUI 发现 Receiver 挂掉了，考虑有没有可能是这个问题。

#### blocksForPushing 
blocksForPushing 作为 currentBuffer 和 BlockManager 之间的中转站。默认存储的数据最大可以达到10*currentBuffer 大小。一般不大可能，除非你的 spark.streaming.blockInterval 设置比10 ms还小，官方推荐最小也要设置成50 ms，你就不要搞对抗了。所以这块不用太担心。

#### blockPushingThread

blockPushingThread 负责从 blocksForPushing 获取数据，并且写入 BlockManager。blockPushingThread 只写自己所在的 Executor 的 blockManager，也就是每个 batch 周期的数据都会被一个 Executor 给扛住了。这是导致内存被撑爆的最大风险。数据量很大的情况下，会导致 Receiver 所在的 Executor 直接挂掉。对应的解决方案是使用多个 Receiver 来消费同一个 topic，使用类似下面的代码：

```
val kafkaDStreams = (1 to kafkaDStreamsNum).map { _ => KafkaUtils.createStream( ssc, zookeeper, groupId, Map("your topic" -> 1), if (memoryOnly) StorageLevel.MEMORY_ONLY else StorageLevel.MEMORY_AND_DISK_SER_2)} val unionDStream = ssc.union(kafkaDStreams) unionDStream
```

#### 动态控制消费速率以及相关论文

前面我们提到，SS 的消费速度可以设置上限，其实 SS 也可以根据之前的周期处理情况来自动调整下一个周期处理的数据量。可以通将 spark.streaming.backpressure.enabled 设置为 true 打开该功能。算法的论文可参考 Socc 2014: Adaptive Stream Processing using Dynamic Batch Sizing，还是有用的，我现在也都开启着。

### 直接方式（无接收器） Direct Approach（No Receivers）

笔者认为，DirectApproach 更符合 Spark 的思维。我们知道，RDD 的概念是一个不变的、分区的数据集合。我们将 Kafka 数据源包裹成了一个 KafkaRDD，RDD 里的 partition 对应的数据源为 Kafka 的 Partition。唯一的区别是数据在 Kafka 里而不是事先被放到 Spark 内存里。

#### Direct Kafka InputDStream

Spark Streaming 通过 Direct Approach 接收数据的入口自然是 KafkaUtils.createDirectStream 了。在调用该方法时，会先创建：

```
val kc = new KafkaCluster(kafkaParams)
```

KafkaCluster 这个类是真实负责和 Kafka 交互的类，该类会获取 Kafka 的 Partition 信息，接着会创建 DirectKafkaInputDStream，每个 Direct Kafka InputDStream 对应一个 Topic。此时会获取每个 Topic 的每个 Partition 的 offset。如果配置成 smallest 则拿到最早的 offset，否则拿最近的 offset。

每个 DirectKafkaInputDStream 也会持有一个 KafkaCluster实例。到了计算周期后，对应的 DirectKafkaInputDStream.compute 方法会被调用，此时做下面几个操作：

- 获取对应 Kafka Partition 的 untilOffset。这样就确定过了需要获取数据的区间，同时也就知道需要计算多少数据了。 

- 构建一个 KafkaRDD 实例。这里我们可以看到，每个计算周期里，DirectKafkaInputDStream和 KafkaRDD 是一一对应的。

- 将相关的 offset 信息报给 InputInfoTracker。

- 返回该 RDD。

#### KafkaRDD 的组成结构

KafkaRDD 包含 N（N=Kafka 的 partition 数目）个 KafkaRDDPartition，每个 KafkaRDDPartition 其实只是包含一些信息，譬如 topic、offset 等，真正如果想要拉数据，是透过 KafkaRDDIterator 来完成，一个 KafkaRDDIterator 对应一个 KafkaRDDPartition。

整个过程都是延时过程，也就是数据其实都在 Kafka 存着呢，直到有实际的 Action 被触发，才会有去 Kafka 主动拉数据。

#### 限速

Direct Approach（NoReceivers）的接收方式也是可以限制接受数据量的。可以通过设置 spark.streaming.Kafka.maxRatePerPartition 来完成对应的配置。需要注意的是，这里是对每个 Partition 进行限速。所以需要事先知道 Kafka 有多少个分区，才好评估系统的实际吞吐量，从而设置该值。相应的，spark.streaming.backpressure.enabled 参数在 Direct Approach 中也是继续有效的。

### 两种方式对比

经过上面对两种数据接收方案的介绍，我们发现，Receiver-based Approach 存在各种内存折腾，对应的 Direct Approach（No Receivers）则显得比较纯粹简单些，这也给其带来了较多的优势，主要有如下几点：

- 因为按需拉数据，所以不存在缓冲区，就不用担心缓冲区把内存撑爆了。这在 Receiver-based Approach 就比较麻烦，需要通过 spark.streaming.blockInterval 等参数来调整。

- 数据默认就被分布到了多个 Executor 上。Receiver-based Approach 需要做特定处理，才能让 Receiver 分不到多个 Executor上。

- Receiver-based Approach 的方式，一旦你的 Batch Processing 被 delay 了，或者被 delay 了很多个 batch,那估计你的 Spark Streaming 程序离崩溃也就不远了。Direct Approach（No Receivers）则完全不会存在类似问题。就算你 delay 了很多个 batch time，你内存中的数据只有这次处理的。

- Direct Approach（No Receivers）直接维护了 Kafka offset,可以保证数据只有被执行成功了，才会被记录下来，透过 checkpoint 机制。如果采用 Receiver-based Approach，消费 Kafka 和数据处理是被分开的，这样就很不好做容错机制，比如系统宕掉了。所以你需要开启 WAL，但是开启 WAL 带来一个问题是，数据量很大，对 HDFS 是个很大的负担，而且也会对实时程序带来比较大延迟。

我原先以为 Direct Approach 因为只有在计算的时候才拉取数据，可能会比 Receiver-based Approach 的方式慢，但是经过我自己的实际测试，总体性能 Direct Approach 会更快些，因为 Receiver-based Approach 可能会有较大的内存隐患，GC 也会影响整体处理速度。

### 如何保证数据接受的可靠性

SS 自身可以做到 at least once 语义，具体方式是通过 Checkpoint 机制。

#### CheckPoint 机制

DStreamGraph 类负责生成任务执行图，而 JobGenerator 则是任务真实的提交者。任务的数据则来源于 DirectKafkaInputDStream，checkPoint 一些相关信息则是由类 DirectKafkaInputDStreamCheckpointData 负责。

先看看 checkpoint 都干了些啥，它其实就序列化了一个类而已：

```
org.apache.spark.streaming.Checkpoint
```

看看类成员都有哪些：

```
val master = ssc.sc.master val framework = ssc.sc.appName val jars = ssc.sc.jars val graph = ssc.graph val checkpointDir = ssc.checkpointDir val checkpointDuration = ssc.checkpointDurationval pendingTimes = ssc.scheduler.getPendingTimes().toArray val delaySeconds = MetadataCleaner.getDelaySeconds(ssc.conf) val sparkConfPairs = ssc.conf.getAll
```

其他的都比较容易理解，最重要的是 graph，该类全路径名是：

```
org.apache.spark.streaming.DStreamGraph
```

里面有两个核心的数据结构是：

```
private val inputStreams = new ArrayBuffer[InputDStream[_]]() private val outputStreams = new ArrayBuffer[DStream[_]]()
```

nputStreams 对应的就是 DirectKafkaInputDStream 了。
再进一步，DirectKafkaInputDStream 有一个重要的对象：

```
protected[streaming] override val checkpointData = new DirectKafkaInputDStreamCheckpointData
```

checkpointData 里则有一个 data 对象，里面存储的内容也很简单：

```
data.asInstanceOf[mutable.HashMap[Time, Array[OffsetRange.OffsetRangeTuple]]]
```

就是每个 batch 的唯一标识 time 对象，以及每个 KafkaRDD 对应的 Kafka 偏移信息。就是每个 batch 的唯一标识 time 对象，以及每个 KafkaRDD 对应的的 Kafka 偏移信息。

而 outputStreams 里则是 RDD，如果存储的时候做了 foreach 操作，那么应该就是 ForEachRDD 了，它被序列化的时候是不包含数据的。

而 downtime 由 checkpoint 时间决定，pending time 之类的也会被序列化。

经过上面的分析，我们发现：

- checkpoint 是非常高效的。没有涉及到实际数据的存储。一般大小只有几十 K，因为只存了 Kafka 的偏移量等信息。

- checkpoint 采用的是序列化机制，尤其是 DStreamGraph 的引入，里面包含了可能如 ForeachRDD 等，而 ForeachRDD 里面的函数应该也会被序列化。如果采用了 CheckPoint 机制，而你的程序包做了变更，恢复后可能会有一定的问题。

接着我们看看 JobGenerator 是怎么提交一个真实的 batch 任务的，分析在什么时间做 checkpoint 操作，从而保证数据的高可用：

- 产生 jobs；

- 成功则提交 jobs 然后异步执行；

- 失败则会发出一个失败的事件；

- 无论成功失败，都发出 DoCheckpoint 事件；

- 任务运行后，会再调用一次 DoCheckpoint 事件。

只要任务运行完成后没能顺利执行完 DoCheckpoint 前 crash，都会导致这次 Batch 被重新调度。也就说无论怎样，不存在丢数据的问题，而这种稳定性是靠 checkpoint 机制以及 Kafka 的可回溯性来完成的。

#### 业务需要做事务，保证 Exactly Once 语义

这里业务场景被区分为两个：

- 幂等操作

- 业务代码需要自身添加事物操作

所谓幂等操作就是重复执行不会产生问题，如果是这种场景，你不需要额外做任何工作。但如果你的应用场景是不允许数据被重复执行的，那只能通过业务自身的逻辑代码来解决了。

这个 SS 倒是也给出了官方方案：

```
dstream.foreachRDD { (rdd, time) => rdd.foreachPartition { partitionIterator => val partitionId = TaskContext.get.partitionId() val uniqueId = generateUniqueId(time.milliseconds, partitionId) // use this uniqueId to transactionally commit the data in partitionIterator } }
```

这代码啥含义呢？就是说针对每个 partition 的数据，产生一个 uniqueId，只有这个 partition 的所有数据被完全消费，则算成功，否则算失败，要回滚。下次重复执行这个 uniqueId 时，如果已经被执行成功过的，则 skip 掉。
这样，就能保证数据 Exactly Once 语义。

### 总结

根据我的实际经验，目前 Direct Approach 稳定性感觉比 Receiver-based Approach 更好些，推荐使用 Direct Approach 方式和 Kafka 进行集成，并且开启相应的 checkpoint 功能，保证数据接收的稳定性，Direct Approach 模式本身可以保证数据 At Least Once 语义，如果你需要 Exactly Once 语义时，需要保证你的业务是幂等，或者保证了相应的事务。