## Spark Streaming 构建有状态的可靠流式处理应用

文/毛玮

>大数据领域里的流式数据处理，是指针对流式数据的一种分布式、低延时、高可用并支持容错的实时计算技术。Spark Streaming 正是目前业界应用最广泛的流式处理引擎之一，它以 Micro Batch 的形式提交作业，并且可以保证 Exactly Once 语义。在流式数据处理场景中，数据生成是动态的、连续不断的，很多时候我们需要结合前一轮作业迭代的结果和当前输入的数据，输出新的迭代结果，这中间就会牵涉到对状态的管理和维护。虽然 Spark Streaming 原生支持基于状态的操作符，但是想要把它用对、用好，并且灵活实现相对复杂的业务逻辑却并不容易，下面笔者就为大家一一道来。

### mapWithState

早在 Spark 0.7版本中，Spark Streaming 就已经有了基于状态的操作符 updateStateByKey。根据用户实际使用中反馈的情况，可以发现 updateStateByKey 存在以下问题：

- 每个批次的状态更新实质上就是两个 RDD 之间的 CoGroup，即使只更新了状态集的某几项，也需要进行全状态集扫描，因此大大降低了效率。当内存中的管理状态集越来越大时，updateStateByKey 操作符的性能就会明显呈现线性下降。 

- 每个批次都必须返回状态全集，不能自定义输出。很多情况下，用户只关心该批次输入数据的相关状态。

- 缺少 timeout 机制，内存中管理的状态集就会不断增大，一些过期不用的状态集则无法回收——这对于一些需要长时间持续运行的流式数据处理来说尤其重要。

针对这些问题，Spark1.6版本特别加强了流式数据的状态管理能力，提出新的操作符 mapWithState。对比原来的 updateStateByKey 操作符，mapWithState 的使用更加灵活。同时据社区提供的测试数据显示，新的操作符在性能上提升了10倍。

下面通过 Spark 源代码给出的 Stateful NetworkWordCount 例子，了解 mapWithState 实现的具体细节。从该代码片段可以看到，区别于 updateStateByKey，mapWithState 不会直接传入 mappingFunc 而是会生成一个 StateSpec 实例。StateSpec 作为一个 case class，能够封装所有和 mapWithState 操作相关的规格说明，包括初始状态、数据分区、超时阈值等。最终，mapWithState 操作符还会生成一个 MapWithStateDStream。

```
// 创建映射函数
val mappingFunc = (word: String, one: Option[Int], state: State[Int]) => {
  val sum = one.getOrElse(0) + state.getOption.getOrElse(0)
  val output = (word, sum)
  state.update(sum)
  output
}
val stateDstream = wordDstream.mapWithState(
  StateSpec.function(mappingFunc).initialState(initialRDD))
```

**代码1：StatefulNetworkWordCount.scala 代码片段**

compute 方法是所有 DStream 里最核心的函数，描述了 DStream 在给定时间点会生成怎样的 RDD。对于 MapWithStateDStream，它的 compute 是委托给内部成员变量 InernalMapWithStateDStream 的 compute 方法，具体如下：

```
override def compute(validTime: Time): Option[RDD[MapWithStateRDDRecord[K, S, E]]] = {
  // 得到上一个批次的状态
  val prevStateRDD = getOrCompute(validTime - slideDuration) match {
    ......
  }
  // 从依赖的DStream得到当前批次的输入数据
  val dataRDD = parent.getOrCompute(validTime).getOrElse {
    context.sparkContext.emptyRDD[(K, V)]
  }
  // 对输入数据重新进行划分
  val partitionedDataRDD = dataRDD.partitionBy(partitioner)
  val timeoutThresholdTime = spec.getTimeoutInterval().map { interval =>
    (validTime - interval).milliseconds
  }
  // 生成MapWithStateRDD并返回
  Some(new MapWithStateRDD(
    prevStateRDD, partitionedDataRDD, mappingFunction, validTime, timeoutThresholdTime))
}
```

**代码2：MapWithStateDStrream.scala 代码片段**

这里最终生成了一个 MapWithStateRDD，内容包含有之前维护的状态（prevStateRDD）、当前输入的数据（dataRDD）、状态映射函数（mappingFunction）、当前 Batch 时间（validTime），以及计算得出的超时阈值时间（timeoutThresholdTime）。此外，MapWithStateRDD 的每个 partition 中都有一个 StateMap 和 mappedData，前者用来存储状态信息，后者用来保留输出结果。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a28609c1e8.png" alt="图1  MapWithStateRDD结构示意图" title="图1  MapWithStateRDD结构示意图" />

**图1 MapWithStateRDD 结构示意图**

每次新生成的 MapWithStateRDD 都会遍历？上 Batch 里的所有 StateMap，并在其中复制状态。StateMap 的代码复制片段如下：

```
def createFromRDD[K: ClassTag, V: ClassTag, S: ClassTag, E: ClassTag](
    rdd: RDD[(K, S, Long)],
    partitioner: Partitioner,
    updateTime: Time): MapWithStateRDD[K, V, S, E] = {
  val pairRDD = rdd.map { x => (x._1, (x._2, x._3)) }
  val stateRDD = pairRDD.partitionBy(partitioner).mapPartitions({ iterator =>
    // 对每一个partition新建一个空的StateMap
val stateMap = StateMap.create[K, S](SparkEnv.get.conf)
// 遍历现有的状态集，一把状态依次放入新建的的StateMap
    iterator.foreach { case (key, (state, updateTime)) =>
      stateMap.put(key, state, updateTime)
    }
    Iterator(MapWithStateRDDRecord(stateMap, Seq.empty[E]))
  }, preservesPartitioning = true)
  // 建立空的输出数据和映射函数
  val emptyDataRDD = pairRDD.sparkContext.emptyRDD[(K, V)].partitionBy(partitioner)
  val noOpFunc = (time: Time, key: K, value: Option[V], state: State[S]) => None
  // 生成新的MapWithStateRDD
  new MapWithStateRDD[K, V, S, E](
    stateRDD, emptyDataRDD, noOpFunc, updateTime, None)
}
```

**代码3：MapWithStateRDD.scala 代码片段**

### 应用案例

说了这么多，下面来看一个实际应用案例。某电商客户希望对其现有的线上流式数据处理任务进行改进和优化，就在一个在线商品实时推荐的典型场景中尝试使用了最新的 mapWithState 操作符来维护状态，同时保证整个应用的高可用性。

对于一个在线购物网站而言，最重要的一环就是针对不同用户提供商品推荐，因此合适的推荐算法可以大大提高网站的成交量，带来巨大的经济效益。然而现在的推荐算法五花八门，该如何做出最好的选择呢？业界就该问题并没有明确答案，因此需要结合实际情况进行分析。客户并没有采用某种单一的推荐算法，而是给不同的推荐算法分配权重，根据权重分配流量给不同的算法，类似于 AB Test 思想。然后再根据用户行为（页面访问次数，订单转化率等），动态化地实时调整各算法权重。 

在这个例子中有两个输入流：

- Stream 1：描述用户分组信息。每一条输入记录包含了用户ID（user\_id），用户被分配到的分类算法种类（rank\_type），以及用户所属人群分类（group\_type）。

- Stream 2：用户页面访问的日志流。从每条记录中可以解析出用户ID（user\_id），及页面打开时间（pageStartTime）等。

当用户的分组信息流入应用后，可以把 user\_id 作为 Key，rank\_type 和 group\_type 作为 Value，在内存中维护用户分组信息的对照表，其中描述了不同用户会被分到的不同推荐算法。每当有新的数据从 Stream 1流入，就对用户分组状态进行更新。接着，将 Batch 间隔时间流入的数据集与最新的用户分组状态和 Stream 2进行 join，这样就可以得到含有分组信息的页面访问数据。然后，把推荐的算法种类和时间窗口聚合可以统计出访问次数。最后，再把聚合结果输出到数据库，其他模块就可以从数据库中获取最新数据，调整不同推荐算法的权重。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2982e7779.png" alt="图2  数据处理逻辑示意图" title="图2  数据处理逻辑示意图" />

**图2  数据处理逻辑示意图**

关于用户分组信息状态的维护，原来使用的是一个额外 stateRDD。每当有数据更新，就将之前的 stateRDD 和更新的数据进行 union，得到新的 stateRDD。每过10个 Batch 程序，就会对该 RDD 做 checkpoint 以避免依赖链过长。代码示例如下：

```
var stateRDD = ssc.sparkContext.parallelize(dummyAbt)
// 从Kafka接收数据
val abtDStream = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, abtParams.toMap, abtTopics)
 
abtDStream.map(log => AbtParser.parse(log)).foreachRDD( (abtRDD, time) => {
  // 保留之前的状态
  val preStateRDD = stateRDD
  ......
   
  // 更新状态集，新的状态集等于之前的状态union上新加入的状态
  // 之后再做reduceByKey操作，对Key重复的状态只保留最新的值
  stateRDD = stateRDD.union(
unionDF.rdd.map(row=>
  (row.get("user_id"),(row.get("rank_type"),"dummy_user_group")))
    ).reduceByKey((a,b)=> b))
  stateRDD.persist()
  ......
  // 每过checkpointInterval的时间间隔，对stateRDD做一次checkpoint，来打断过长的依赖链
  if (time.milliseconds % checkpointInterval == 0) {
    stateRDD.checkpoint()
  }
  ......
  // 显式的把之前的状态集从cache里清除
  preStateRDD.unpersist()
}
```

**代码4：使用额外 stateRDD 代码片段**

但这种做法也存在着问题：

1. stateRDD 是一个额外 RDD，在 DStreamGraph 感知之外，程序重启之后并不能从 checkpoint 中恢复，换句话说也就是不支持高可用性；

2. 运行一段时间后 stateRDD 数据会聚集到1或2个的 Executor 上并成为热点，不仅会影响之后迭代的资源分配，还会降低系统的计算效率和稳定性；

3. 代码逻辑复杂，需要额外的显式 checkpoint 来打断 RDD 的 lineage；

4. stateRDD 产生的 checkpoint 文件不会自动回收，导致文件越来越多。

改用 mapWithState 则能够很好地解决这些问题，并实现相同的业务逻辑。一般情况下，Spark Streaming 框架会在10个 Batch 后，对 mapWithState 操作符产生的 RDD 做 checkpoint，并且自动回收不需要的 checkpoint 文件。状态信息维护在整个 DStreamGraph 处理流内部，不必依赖额外的 RDD。

```
// 得到当前批次的输入数据
val validAbtDStream = 
  abtDStreams.map(log => AbtParserNew.parseNew(log._2)).transform((abtRdd, time) => {
    val sqlContext = HiveContextSingleton.getInstance(abtRdd.sparkContext)
    import sqlContext.implicits._
......
 
val abtMidRdd = midMergedDF.rdd.map(row => 
  row.get("user_id"),(row.get("rank_type"),"dummy_user_group"))
    abtMidRdd
})
// 创建状态映射函数，对状态进行更新
val stateFunc = (id: String, value: Option[(String, String)], state: State[(String, String)]) => {
  if (value.isDefined)
    state.update(value.get)
  id
}
// 调用mapWithState并传入状态映射函数，stateDStream是输出数据对应的DStream，里面包含了状态集
val stateDStream = validAbtDStream.mapWithState(StateSpec.function(stateFunc))
// 调用stateSnapShots函数，得到状态集对应的DStream
stateDStream.stateSnapshots()
```

**代码5：改用 mapWithState 后的代码片段**

### 处理数据延迟

还是上面的例子，由于 Stream 1和 Stream 2不同源，它们的相关性并不能得到很好的保证。举例来说，Stream 2的数据已经显示用户123访问了某页面，但是用户分组信息来的慢了，用户123的分组信息就会被划分到下一个 Batch。这时如果要对 Stream 2数据和 Steam 1数据进行 Join 就会导致用户123信息不匹配。这种数据延时的情况，在实际业务逻辑中十分常见。

遗憾的是，Spark Streaming 完全按照事件到达时间来划分作业，本身并不支持按事件产生时间（event time）来划分作业，也没有类似 Watermark 的机制来解决上述问题。那是不是说 Spark Streaming 对于这种情况就无能为力了呢？下面就提供两种思路来解决这个问题。

#### 第一种方法：使用 mapWithState 和最新支持的 timeout 机制

把 Stream 2当前的 Batch 值和 Stream 1维护状态进行匹配，如果匹配不上就套用 mapWithState 函数，更新未匹配的数据状态，然后把过程中没有匹配上的数据集再次和 Stream 1分组状态进行匹配。然后再把两次匹配成功的项 union 起来，得到最终结果。这种方法的好处是用户可以随意设置自己想要的 timeout：假设设置成1分钟就表示如果一条记录1分钟内还没有匹配上就丢弃。缺点的话，对比下面要说到的另一种 window 使用方法，内存上会有额外消耗，导致运行效率有所降低；同时 mapWithState 的 timeout 机制也有很多坑，用起来有很多需要注意的地方（后面会详细说说 timeout 里的坑）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2a4031e33.png" alt="图3  逻辑示意图" title="图3  逻辑示意图" />

**图3 逻辑示意图**

这里也提供一份简短的示例代码：

```
// 目标匹配数据集
val targetWords = mutable.Set[String]("a", "b", "c")
val pvDStream = ssc.socketTextStream("localhost", 9999).map(word => (word, 1))
// 匹配上的数据流
val matchedDStream = pvDStream.filter{
  case (word, _) => targetWords.contains(word)
}
// 未匹配上的数据流
val unmatchedDStream = pvDStream.filter{
  case (word, _) => !targetWords.contains(word)
}
 
val secondChanceFunc =(word: String, one: Option[Int], state: State[Int]) => {
  if (one.isDefined) {
    state.update(1)
  }
  if(state.isTimingOut()) None else Some(word)
}
 
val stateDStream = unmatchedDStream.mapWithState(StateSpec.function(secondChanceFunc)
  .timeout(batchInteval))
// 默认checkpoint的时间间隔是Batch时间间隔的10倍，这里设置成每个Batch都需要做checkpoint.
stateDStream.checkpoint(batchInteval)
val secondChanceDStream = stateDStream.stateSnapshots()
// 新匹配成功的数据流
val secondChanceMatchedDStream = secondChanceDStream.filter {
  case (word, _) => targetWords.contains(word)
}
// 对两部份匹配成功的数据流进行Union得到最后的结果
val resultDStream = matchedDStream.union(secondChanceMatchedDStream)
```

**代码6： 使用 mapWithState 来处理数据延迟的示例代码**

#### 第二种方法：使用 window 操作符

这种方法在主要逻辑上和第一种方法大同小异，但在使用 mapWithState 的地方代替使用了 window 操作符，在保持计算频度不变的同时，把计算覆盖到的数据集范围放大。和前一种方法相比，它只能给那些没有匹配上的数据一次机会，在下一个 Batch 上进行匹配。但是如果相关的流数据事件是在几个 Batch 之后才到达，这个方法会有较大限制。因为当 window 窗口大于两个 Batch 长度时，部分数据会被重复计算，可能会影响正确性。在上面的案例中，客户把匹配成功的数据写到一个幂等的（idempotent）外部储存中去，可以容忍数据的重复计算而不影响正确性，所以最终使用了这种方法。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2b2b15a29.png" alt="图4  逻辑示意图" title="图4  逻辑示意图" />

**图4  逻辑示意图**

### 踩过的那些坑

#### 第1坑：mapWithState 里的 timeout 机制

就像前面提到过的，当某条状态信息的 timeout 阈值到了之后该条状态信息并不会马上从状态表里删除，只是会标记成 isTimeout=true 的状态，这时便不能再对该条记录进行修改，否则就会出现异常。具体的删除操作是在要做 checkpoint 的 Batch 中被执行的（默认 mapWithState 每运行10个 Batch 会做一次 checkpoint）。同时，MapWithStateRDD 还会做对所有状态做 full scan 操作，所以一旦使用了 timeout 机制，在写 mappingFunc 的时候要格外注意：在对 State 进行 update 前，需要检查一下 Option[Value]的值是否为 None，当不为 None 的时候才能做 update.

还有一个问题，timeout 被删除的数据并不是从状态表里默默被删除就结束了，而是会再次执行 mappingFunc，把数据放到待输出的数组里和原有结果一起输出，因此会导致用户无法区分哪些是 timeout 输出，哪些是原有输出。具体源代码如下：

```
if (removeTimedoutData && timeoutThresholdTime.isDefined) {
  // 得到状态集里所有跟新时间在timeoutThresholdTime之前的状态项
  newStateMap.getByTime(timeoutThresholdTime.get).foreach { case (key, state, _) =>
wrappedState.wrapTimingOutState(state)
     // 把超时的状态项进行数据映射
    val returned = mappingFunction(batchTime, key, None, wrappedState)
    // 把得到的结果写入到输出结果里去    
    mappedData ++= returned
    // 从状态集里删除状态项
    newStateMap.remove(key)
  }
}
```

**代码7： 状态删除代码片段**

#### 第2坑：mapWithState 和 Window 的结合导致 OOM

对新代码进行测试时会发现：程序在持续运行一段时间后，Executor 端就会频繁地 Full GC，导致每个 Batch 的 Process Time 越来越长，最终发生 OOM。这里的代码逻辑是先用 mapWithState 维护 Stream 1状态，一系列操作后再接上一个 window 操作符把每个 Bacth 时间间隔从30秒延长到5分钟，然后再次接上一个 mapWithState 操作符保留输出前状态，最后输出到数据库。通过 Spark WebUI 可以观察到，RDD 运行8小时后，它的维护状态表大小已经接近了500 M，同时这个状态在内存里还被 cache 了280多份。因为程序中 rememberDuration 时间被默认为 checkpointDuration 的2倍，而 checkpointDuration 默认是 batchInterval 的10倍，也就是说在上面的逻辑里第一次 BatchInterval 的数据被 cache 了100分钟。每30秒生成一次数据，所有内存里至少也有200+份状态是 GC 不掉的。解决方法就是手动调小 checkpointDuration 和 rememberDuration，不过这样也会让 checkpoint 变得频繁，运行效率有所损失。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2c18ee352.png" alt="图5  Spark UI: Storage Tab" title="图5  Spark UI: Storage Tab" />

**图5 Spark UI: Storage Tab**

#### 第3坑：SqlContext 不能从 checkpoint 恢复的问题

用 mapWithState 主要是为了实现高可用性，但在替换了原有逻辑之后仍然会发现：重启后会出现异常，SqlContext 值为空。这里主要是因为程序里用了 Spark SQL，而 SqlContext 的信息并不会保存在 checkpoint 里，重启的时候没有被正确的初始化。可以用以下方法绕过去：建立一个 SqlContextSingleton，在用 SQLContext 的时候调用 SqlContextSingleton 里的 getInstance 来获取。这样重启之后由于 instance 的值为空，就会重新触发一次初始化动作，使程序正常重启。当前整个 Streaming 项目都在往 SQL 上调整，相信 Structured Streaming 出来以后这个问题会有更好的解决方案。

```
object HiveContextSingleton {
  @transient private var instance :  HiveContext = _
  //如果instance有值直接返回当前值，如果为空则根据传进来的SparkContext重新初始化SQLContext
  def getInstance(sc : SparkContext) : HiveContext = synchronized {
    if(instance == null) {
      instance = new HiveContext(sc)
    }
    instance
  }
}
 
DStream.foreachRDD(rdd => {
  // 调用HiveContextSingleton来拿到当前的SQLContext
  val sqlContext = HiveContextSingleton.getInstance(rdd.sparkContext)
  import sqlContext.implicits._
     
  ...
})
```

**代码8：SqlContextSingleton 代码示例**

### 测试结果

目前，新的代码已经正式上线。使用 mapWithState 操作符之后，客户最关心的的程序高可用性得到了保证，在处理效率和内存使用上也较原来的代码有了显著提升。为保持集群配置和原来相同，还给应用程序分配了18个 Executor，每个 Executor 占用12个 Core 和24 G 的 Memory。这样运行8小时后仅仅使用了30 G 的内存，占总 Cache 可使用内存的1/10，并且基本稳定在这一水平，没有出现明显增长。
 
<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2cf6b0012.png" alt="图6 Spark UI: Executor Tab" title="图6 Spark UI: Executor Tab" />

**图6 Spark UI: Executor Tab**

关于前面提到的一点——状态信息 Partition 的分布会发生数据倾斜，在改用 mapWithState 之后也已经解决。通过下图可以看到：现在 RDD 的各个 Partition 分布相对均匀，16个 Partition 分布在9个 Executor 上，且每个 Executor 至多只有2个 Partitions。这样分布，对于充分利用集群的网络带宽和计算资源是非常有好处的。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2d2522aa3.png" alt="图7 Spark UI: Partition分布信息" title="图7 Spark UI: Partition分布信息" />

**图7 Spark UI: Partition 分布信息**

此外，程序处理时间和 Total Delay 也基本稳定在了30秒，远小于 Batch Interval 设置的5分钟。相比之下，原来的代码在长时间运行后几乎会占用所有的内存，Delay 越变越长，导致需要定时重启才能保持程序稳定运行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2d4db2031.png" alt="图8  Spark UI: Streaming Tab" title="图8  Spark UI: Streaming Tab" />

**图8 Spark UI: Streaming Tab**

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577a2d762e2f7.png" alt="图9  Spark UI: Streaming Tab" title="图9  Spark UI: Streaming Tab" />

**图9 Spark UI: Streaming Tab**