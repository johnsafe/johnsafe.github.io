## Spark Streaming 在腾讯广点通的应用

文/林立伟

>Spark Streaming 可以从多种数据源获取实时输入，经过实时处理，再把数据写到不同的数据存储中。在实时计算过程中，还可以与 Spark SQL、MLLib 等进行交互。本文将分享 Spark Streaming 在广点通的实践总结。

腾讯社交广告是基于整个腾讯社交体系的广告平台，覆盖了微信、QQ 客户端、手机 QQ、腾讯新闻客户端等流量，推广形式包括 Banner、插屏、开屏、信息流广告等。广点通是腾讯社交广告的技术品牌。

### Spark Streaming 基本结构及在广点通的应用概述

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1552e7c2e.png" alt="图1  基于Spark Streaming实时数据处理架构" title="图1  基于Spark Streaming实时数据处理架构" />

**图1 基于 Spark Streaming 实时数据处理架构**

Spark Streaming 可以从多种数据源获取实时输入，经过实时处理，再把数据写到不同的数据存储中。在实时计算过程中，还可以与 Spark SQL、MLLib 等进行交互。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b15a0ccf1c.png" alt="图2  Spark Streaming底层数据传输流程" title="图2  Spark Streaming底层数据传输流程" />

**图2 Spark Streaming 底层数据传输流程**

稍微往下层看一点，实现层面上，Spark Streaming 可以在 WorkerNode 起长时间运行的 Receiver 来进行数据接收，然后在 driver 端对接收到的数据进行信息汇总，并依靠下层 Spark 提供的 SparkContext 下发数据计算任务到 Worker Node，再由 Worker Node 进行真正的数据处理，并把结果写出。

在腾讯内部，负责全公司基础平台的数据平台部为我们提供了如表1所示的 Spark 技术栈。

**表1  Spark 技术栈**

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b15c4a6656.png" alt="表1  Spark技术栈" title="表1  Spark技术栈" />

具体到在广点通，正式线上运行的 Spark Streaming 应用数情况如表2总结。

**表2  广点通应用情况**

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1639028fb.png" alt="表2  广点通应用情况" title="表2  广点通应用情况" />

### Spark Streaming 的特性及应用

本节主要叙述 Spark Streaming 的3个特性，以及这些特性在广点通相应的应用情况。

#### 特性一：Exactly-once 语义

我们知道，Spark Streaming 是基于 Spark 的执行引擎，在 Spark 里，基本的执行单元是任务（task），一个 task 对应处理一批数据，因此这批数据要么全部处理成功，要么全部处理失败。对于一批5条数据，我们会经过中间处理将其扩展为10条，那么这个 task 最终的结果输出就应该是10条。但是这里也存在一些特殊情况：首先是 task 重做，如 task 做到一半失败了，那么就会另外起一个 task 重做，但是最终输出的数据仍然需要保证10条；另一个情况是推测执行，这时对着同一批5条数据，就会有两个 task 同时做，但即使两个 task 都做完，各产生了10条数据，最终仍然只能有一个 task 的10条数据提交成功。在这里，Spark 推测执行的语义与 MapReduce 一致。

对比其他系统，Storm 是 at-most-once 或 at-least-once，MapReduce 是 exactly-once。需要说明的是，这里 exactly-once 是指最终输出，在 task 执行过程中可能会产生 at-least-once 的副作用（因为可能涉及 task 重做）。

针对 Spark Streaming 的 exactly-once 语义，广点通有一些对应的应用。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1bce56801.png" alt="图3  准确实时的数据转移pipeline" title="图3  准确实时的数据转移pipeline" />

**图3 准确实时的数据转移 pipeline**

**应用一：准确实时的数据转移 pipeline**

实际场景中，我们会从 HDFS、TDBank 等数据源取得数据，随后进行数据的清洗、去重、外部数据的查询和当前数据的补全、转换，最终会写出到 HDFS。这里由于是 exactly-once 的语义，那么最终写出到 HDFS 的数据条数与输入 pipeline 的数据条数必然是一致的，即整个数据转移的 pipeline 处理非常准确和实时。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1be75922f.png" alt="图4  反作弊和计费相关等重要业务" title="图4  反作弊和计费相关等重要业务" />

**图4 反作弊和计费相关等重要业务**

**应用二：反作弊和计费相关等重要业务**

这里的 pipeline 从数据源取得数据，同样会进行清理、外部数据的查询和补全，然后进入反作弊流程，再接着是计费流程，最后是做一些策略，并把明细写出。这里可以看到，同样由于 Spark Streaming 提供了 exactly-once 语义，反作弊、计费这样重要的业务流程才可能通过其实现。而在生产环境中，这些 pipeline 也实现了长时间无状况运行。

#### 特性二：可靠状态管理

本节主要关注 Spark Streaming 提供的可靠状态管理特性。Spark Streaming 提供了面向状态的（状态即数据）原生支持，这主要依赖于 Spark 的最基本数据抽象 RDD（Resilient Distributed Datasets），其自身即是可恢复的数据集。具体来看，多个 RDD 会形成一个世系 lineage，因此 RDD 数据失效后则可通过向前回溯进行一个重做，来完成失效数据的恢复。而在具体设施过程中，这里还可通过对通路中某些 RDD 做 checkpoint 来加快整个通路的数据恢复过程。基于 Spark 的 RDD 的数据失效恢复，Spark Streaming 提供了可靠的状态管理：

- 1.5 版：updateStateByKey

- 1.6 版：key value 形式的 mapWithState

- 2.0-SNAPSHOT：更为优化 key value 形式的 State Store

利用 Spark Streaming 可靠状态管理的特性可以做很多跨 batch 的聚合应用，如 pv/uv 计算用例，即一段时间内记录的去重，还包括微额记账等。

**微额记账应用背景**

在广点通系统里，有一部分计费是微额的，即每一条数额特别小，不产生扣费，需要累积并达到特定金额后合并扣费。最原始的 in-house 实现存在过期清理的问题，尚未达到扣费的部分会被舍弃，导致了一些收入流失。后来改进版本解决了过期清理问题，但扩展性不是特别好。再后来是通过 Spark Streaming 实现，其可靠状态管理能够非常好的满足微额记账的应用需求，具体过程参见图5。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1c2153919.png" alt="图5  微额记账应用数据聚合流程" title="图5  微额记账应用数据聚合流程" />

**图5  微额记账应用数据聚合流程**

向第1个 batch 里输入数据，并在 batch 内聚合得到一个汇总结果，随即把这个汇总结果写出到 HDFS。在第2个 batch 里同样是先 batch 内进行聚合，然后与前1个 batch 的聚合结果进行二次聚合，得到综合2个 batch 聚合之后的结果并写到 HDFS。如此一直进行下去，到第60个 batch 则得到跨60个 batch 的全部聚合结果。
在具体实现中，这里定义了一个 DStream，完成先写出数据到 HDFS，随后再进行下一个 batch 聚合的过程。利用这种跨 batch 聚合的方法，微额记账应用具备了非常好的扩展性，这里累积一天、一周、乃至一个月的聚合数据，一旦达到扣费标准，即随时进行相应的扣费。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1c4585a02.png" alt="图6  Spark 2.0版本基于Structured Streaming的微额记账" title="图6  Spark 2.0版本基于Structured Streaming的微额记账" />

**图6  Spark 2.0版本基于 Structured Streaming 的微额记账**

其实类似微额记账这种跨 batch 聚合的需求非常大，所以在 Spark 2.0版本的 Structured Streaming 里，这个聚合的状态被抽象为 key-value 形式的 State Store，其数据被保存到了内存以提高性能；同时，每个 batch 结束前，保存在内存的状态会被持久化到 HDFS。在下一个 batch 计算时，首先会从内存取出这部分状态，并经过一个增量执行（incremental execution），与当前 batch 的数据进行一个增量的聚合，得到一个新的跨 batch 状态。另外，为了加速写 HDFS 这个过程，每次 State Store 刷出状态的时，只会写一份增量数据 delta，并不刷出全量数据。在这里，会有后台线程不断进行全量数据和增量数据的合并，以加快状态故障时的恢复过程。
由此，通过这种 State Store+incremental execution 的方式，跨 batch 聚合的多种应用就得到了很好的支持。

#### 特性三：快速 batch 调度

本节主要关注 Spark Streaming 的快速 batch 调度特性。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1cba011b0.png" alt="图7  不同MapReduce程序之间的调度" title="图7  不同MapReduce程序之间的调度" />

**图7  不同 MapReduce 程序之间的调度**

如果想周期性跑一个 MapReduce 程序，如图6有两个不同的 MapReduce 程序，每个 MapReduce 程序会每隔10分钟周期性的跑一次，这时一般可以引入类似 Ozzie 来进行调度。

但这种情况下是一级的调度，每调度一个实例（如图示有6个实例），都需要与状态 DB 进行交互，包括状态查询、状态更新、依赖重计算等，这时状态 DB 就可能成为瓶颈，对频繁的实例调度进行限制，调度间隔不能太短。比如在腾讯内部，同一 MapReduce 程序的不同实例调度最小间隔是10分钟。另外，一级调度发生后还需要申请资源、下发 jar 包、创建进程等动作，会带来一定的启动时间；也即是说，即使调度完成，计算会等待一定时间后再开始，视不同情况通常有几秒到十几秒的延迟。

再看 Spark Streaming。假设一级的调度系统还是 Oozie，则当 Spark Streaming 程序本身被一级系统 Oozie 调度起来以后，就由 Spark Streaming 的 driver 充当二级调度系统；此时 driver 本身不会与一级调度系统的状态 DB 进行交互，这里后台状态 DB 就不会成为瓶颈。也即 Spark Streaming 自身充当二级调度系统，可以非常快速地调度、下发任务，比如每隔5秒钟或30秒钟就调度一次计算，对比同等情况下 MapReduce 实例每隔10分钟的调度就是一个非常大的改进，是一个非常实时的计算调度。而且，由于 Spark Streaming 进程本身是常驻的，计算只是一个线程粒度的调度，不存在进程创建等等的启动时间，任务被调度后即立刻开始执行。下面看基于快度 batch 调度特性构建的应用。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1cd79b4b0.png" alt="图8  快度batch调度特性应用" title="图8  快度batch调度特性应用" />

***图8 快度 batch 调度特性应用**

在第一个应用里，每30秒或每10秒从多个数据源取得数据作为 input，再对这个 input 进行数据的交叉对比，最后将结果写回到数据存储，或者直接对异常结果进行报警等策略处理。每一个 batch 都做类似的动作；即，这里基于 Spark Streaming 的快速 batch 调度构建了一个关键数据指标的监控、异常数据报警及策略处理的应用。
另外一个应用是，在复杂的 pipeline 里，可能当前 batch 有一小部分数据没有处理成功（原因可能是部分外部数据暂时未就绪，导致这部分数据暂未关联成功），这时可以依靠 driver 的快速 batch 调度，将此部分未成功的数据尽快下发到最近一个未开始的 batch 里，完成这部分数据的快速重试。比如说 batch 1尚未完成时，batch 2已经开始运行，所以当 batch 1一旦完成，就把需要重试的数据交给下一个即将开始的 batch，即 batch 3来完成这部分数据的重试。也即实现了复杂 pipeline 未成功数据的唯一和快速重试。

坦白讲，这里对快速 batch 调度特性的一些应用是把 Spark Streaming 当做一个调度快、运行也快的 MapReduce 在使用。

### Spark Streaming 程序优化经验

前文是 Spark Streaming 的一些特性和广点通的应用，这里将总结 Spark Streaming 程序的一些优化经验。

#### 增加 Memory Back Pressure
Spark Streaming 1.5里增加了 Back Pressure 特性，能够动态调控 Receiver 的接收速率，使整个 Spark Streaming 应用保持稳定。这里在实践中遇到一个极端情况：上游数据源突然产生很大的波动，短时间内产生了大量数据，那么将导致 Receiver 一直无阻拦地接收，最终导致 Out of Memory 溢出。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1cf754f50.png" alt="图9  Executor内存使用及Receiver数据接收" title="图9  Executor内存使用及Receiver数据接收" />

**图9 Executor 内存使用及 Receiver 数据接收**

应对这种极端情况，我们在 Spark Streaming 原有的 Back Pressure 机制基础上增加了 Memory Back Pressure，根据 Executor 的内存使用状况，动态调整 Receiver 的接收速率。如图8，中部小山形状是 Executor 的内存变化，是原因；上面曲线是 Receiver 接收数据情况，是动态响应的结果。可以看到，当 Executor 内存压力非常大时，Memory Back Pressure 产生作用使  Receiver 的接受速率降至很低，对自身进行最大的保护，不再接收更多数据。

此特性主要通过修改 RateLimiter 的源码实现。而且通过下面一条经验，我们即可将其发布到线上生产环境中使用。

#### 最小化发布新特性

我们会经常对 Spark Streaming 原有特性进行修改，也会经常为 Spark Streaming 增加新特性。将这些增加或修改推送到开源社区到发布应用需几个月的时间，此时我们会并行地先在内部环境（在不需要重新 build 整个 Spark 工程的情况下）进行最小化发布，具体的分两种情况。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1d18a9a2c.png" alt="图10  源码修改的两种情况" title="图10  源码修改的两种情况" />

**图10 源码修改的两种情况**

第一种，需要修改的源代码本身行数不多（几百行以内），会直接在原有的源代码上修改，再把这个改过的类，比如刚才的 RateLimiter 打包到自己的应用中。但这里需要遵循原有的代码包结构，例如要放到 src/org/apache/spark/streaming/receiver 包下，然后是原文件名 RateLimiter.scala。此时对整个工程进行打包，上传到 YARN，运行前给 Spark 加参数 spark.driver/executor.userClassPathFirst=false; spark.driver/executor.extraClassPath=app.jar 运行即可，此时新特性就已经发布到了线上环境中。

另一种情况，原始的代码行数就很多，直接修改源代码并不容易，此时可以不用改源代码，而修改编译出来的字节码.class 文件。我们有一个特性是为 Executor 增加一个启动钩子，即在每个 Executor 启动后但尚未执行任何 task 之前，执行一段自定义的代码。此时我们直接修改 Executo 的字节码.class 文件，找到其中需要修改的特定方法（这个方法编译出来的整个字节码只在100行左右），直接为其增加10行左右的字节码（请见图8-B 部分）即完成了增加 Executor 启动钩子的特性。同样的，修改完成之后，也要按照原始的包名进行放置，但注意此次放需放置到 resources 目录下，最后打包、上传，加同样的运行参数。

通过这里的优化方法，开发者就能够快速将 Spark 增加的新特性上线。需要注意，读者也可采用重新 build Spark 工程的方式增加新特性；我们之所以采取这种快速上线的方式，是在发现大数据量下 Spark Streaming 平台暴露的极端问题时，能够第一时间修复并上线，而不需要依赖来自平台支持方的发布（广点通依赖的 Spark 平台是由腾讯另外部门来维护，具体请参见概述部分）。

#### 在 Streaming 程序里，优先使用 DataFrame API

在 Spark Streaming 里，相对使用 RDD API，这里建议优先使用 SparkSQL 提供的 DataFrame API。众所周知，Spark Streaming 提供的 DStream 中的 API 其实暴露出来的是 RDD 的 API，我们可以直接对这个 RDD 进行操作，也可以进一步将其封装成 SparkSQL 的 DataFrame，然后再使用 DataFrame 的 API 进行操作。那么，为什么优先使用 DataFrame API 呢？

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1d3950e88.png" alt="图11  Spark语句优化流程" title="图11  Spark语句优化流程" />

**图11 Spark 语句优化流程**

最主要的因素是，DataFrame API 写出来的程序通常比 RDD API 写出来的程序运行得更快。这里有几个方面的原因。第一个是 DataFrame 会由 Catalyst 优化器把程序输入先转成逻辑计划（Logical Plan），在应用很多逻辑优化后，再转成物理计划（Physical Plan），最后编译为 RDD 去实际执行。这里的逻辑优化包括很多传统的优化方式，比如 Filter-push-down、小表广播优化等，而我们若在 RDD 上手写这些优化是非常麻烦的。再一个原因是，DataFrame API 写出来的程序能够完全跑在 Tungsten Engine 上，后者有很多很厉害的执行层面的优化。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1d54998d4.png" alt="图12  不同类型API 1000万条数据聚合使用时间" title="图12  不同类型API 1000万条数据聚合使用时间" />

**图12 不同类型 API 1000万条数据聚合使用时间**

这里不赘述原理，直接看运行效果，如图12示，对于一个集进行聚合操作，用 Scala 的 RDD API 来写，运行时间是约4秒，用 DataFrame API 来写的话，运行时间就可以缩短到约2秒，整整缩短了一半。运行时，数据集 cache 过程中 DataFrame 比 RDD 内存占用更少。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1d6fb6a68.png" alt="图13  不同API的内存占用" title="图13  不同API的内存占用" />

**图13 不同 API 的内存占用**

如图13所示，同样的数据集，DataFrame 占用空间比 RDD 的1/4还少。也即，给定同等大小的内存，DataFrame 能 cache 更多数据，运行起来进而也更快。既然 DataFrame API 写的程序通常会比 RDD API 写的程序运行得更快，那么如何在 SparkStreaming 里使用 DataFrame API 呢？

其实也很简单。在 Spark 1.x里，我们可以直接在 foreachRDD 里将 RDD 封装为一个 DataFrame：

```
dstream.foreachRDD { rdd => 
  rdd.toDF().select…
}
```

**代码1**

在 Spark 2.0的 Structured Streaming 里，底层执行引擎已经由 RDD 替换为 SparkSQL 了，所以我们直接如下即可：

```
spark.….stream.….startStream()
```

**代码2**

不过另一方面，在现在 1.x 版本里，到底选用 DataFrame API 或者 RDD API 还是应该根据实际应用场景。RDD API 也有一个优点，即它是 Spark 最底层的执行引擎，如果读者对 Spark 底层的执行引擎熟悉，那么 RDD API 写出来的程序就可以非常简单地去推理 RDD 的执行结果，或者去推理 RDD 的执行时间，有问题应该如何改进。

#### task 内按需异步执行

Spark 的数据处理是通过 iterator()方式的进行同步操作，在一 task 里，一条数据进来后，若数据处理本身比较快，但需要进行外部数据查询，那么很多时间就会浪费在网络上。如图示，这个同步线程约有20%时间在计算，80%时间在 sleep 等待外部结果的返回，此时吞吐量就会非常低。应对这种情况，我们使用一个异步执行的 iterator，并配合一个线程池来进行优化。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1d9b438fb.png" alt="图14  不同类型运行效率对比" title="图14  不同类型运行效率对比" />

**图14 不同类型运行效率对比**

具体的，如图14所示，当第1条数据进来，放到第1个线程里；继续去取第2条数据，放到第2个线程里，以此类推。这样，即使在网络上有等待消耗也没有关系，因为此时并不是一个完全同步的串行等待，而是多条数据在同时处理、同时等待。最后，本 task 的 iterator 的结束，是以异步处理的最后一条数据的结束而结束的，也即，即使中间采用了异步执行的实现，但仍然维护了同步 iterator 的语义。在下游任务看来没有任何区别，但吞吐量能够提升数倍。

#### try-catch 保证 Spark Streaming 应用长时运行

我们也听到一些用户调侃 Spark Streaming 应用怎么“跑跑就挂”，其实这不是 Spark Streaming 框架的 bug，而是一个特性。原理是在远程 task 端执行遭遇的错误，会被传递到 driver 端，然后在 driver 端进行抛出。也即是，分布式执行出现了问题，会尽早地经 driver 使得上层应用能够捕捉到这个问题。当然，在 driver 端如果应用不进行捕捉、处理的话，将导致主程序直接暴力地退出，也即整个应用挂掉。其实解决方法很简单，即加一个 try-catch，代码段如下。我们捕捉到一个异常，对其进行处理了就可以了，不会导致主程序挂掉。

```
val inputDStream = ssc.fileStream("...")
inputDStream.foreachRDD(rdd => {
  try {
    // do something
  } catch {
    case NonFatal(e) => errorHandler(e)
  }
})
```

**代码3**

这里有两点需要注意。一是 try-catch 要加到 foreachrdd 里面才产生作用；二是如果用 Scala 来写，要捕捉 NonFatal(e)，而非捕捉 Throwable。

用这种方法就可以屏蔽很多问题，包括 Streamimg 应用会经常遇到的 Could not compute split 等问题。不过我们建议有问题还需要针对性解决，try-catch 只是一个保障长时间运行的屏蔽问题的手段。这里针对 Could not compute split 问题的根本性解决，我们另外在 GitHub 上有专门的 issue 进行了讨论个解决，这里不再赘述。

#### 按需开启 ConcurrentJobs 参数

Spark Streaming 在执行 batch 的时候，默认会串行执行，即前一个 batch 没执行完，后一个 batch 会一直等待，不开始执行。其实这点可以通过内部参数 ConcurrentJobs 进行设置，当设置为 n 的时候，一般可以同时执行 n 个 output，一般也对应着 n 个 batch。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1dc1d6657.png" alt="图15  batch的执行状况" title="图15  batch的执行状况" />

**图15 batch 的执行状况**

如图15里有10个 batch （我们设置了 Concurrent Jobs=5），那么即有5个 batch 在并行运行，其余5个 batch 在排队等待运行。这里如果 Spark Streaming 应用里，batch 之间没有相互依赖，但我们还是推荐设置这个参数的，避免某个 batch 的长时间执行对整个 Streaming 应用产生影响。

#### 通过调优器进行进程调优

最后，我们非常推荐使用自己的 profiler （调优器）针对 Executor的 JVM 进行进程级别的观察和优化。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b1de5129bf.png" alt="图16  profiler视图" title="图16  profiler视图" />

**图16 profiler 视图**

如图16所示，profiler 能够提供 Executor 端的内存占用情况、线程运行情况（大多数时间是在计算还是在等待），哪些类创建了过多对象导致了大量 GC 等等，这些信息都非常有用。我们的经验是，通常相对未调优的程序，通过 profiler 调优能够带来2-5倍的性能改进。
不过实际中使用 profiler 还有一个问题。如果应用是运行在 YARN 上，那么 Executor 的位置和端口每次重新启动应用都会变化；那么在哪台物理节点上安装 profiler 以及 profiler 监听哪个进程就是问题，这里的建议是做到按需分发。

具体过程是把 profiler 本身打包到应用的 jar 里，然后依靠 YARN 应用需要分发 jar 包的过程，随之将 profiler 分发到相应的 Executor 所在节点。我们需要在应用里，在 Executor 启动后、执行任何 task 前，执行一段自己的代码，将 profiler 从应用里解压出来，并在新进程里启动、监听当前相应 Executor 进程。最后，当调试完成应用退出时，profiler 也会随之退出，不会形成孤儿进程。

### 总结与展望

上文我们讨论了 Spark Streaming 的3个特性，包括 exactly-once、可靠状态管理、快速 batch 调度，以及基于这些特性所打造的计费、聚合、监控等方面的应用。

在未来，我们将继续在 Spark Steaming 上做一些更深入的工作。一方我们会迁移更多原本基于 MapReduce 保证离线数据准确，配合 Storm 保证数据快速的 Lambda 架构的应用到 Spark Streaming 上，也即只用 Spark Streaming 一套系统来同时保证数据的准确性和实时性。

另一方面，在 Spark 2.0里，Structured Streaming 将完全基于SparkSQL引擎构建，也引入了 event time、session 等新特性，我们也会将目前的 Streamimg 应用全面迁移到 Spark 2.0的 Structured Streaming 上。