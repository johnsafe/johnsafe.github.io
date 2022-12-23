## Spark Streaming 在猎豹移动的实践

文/朱卫斌

>作者所在团队主要负责猎豹移动的数据平台 Infoc，核心职责主要有数据仓库和实时计算平台（后文简称实时平台）两个。其中，实时平台需要支持每日数十亿级数据量的实时计算需求。在团队中，笔者主要负责及参与：实时平台从无到有的建立、实时平台整体框架的设计与实现、实时平台周边基础设施的完善，以及实时需求的开发与维护。本文将详细介绍我们基于 Spark Streaming 的实时计算实践。

### 实时计算平台的新老交替

猎豹移动一直以来都有用户量非常大的产品，从 PC 端的金山毒霸、金山卫士、猎豹浏览器等，到移动端的猎豹清理大师、猎豹安全大师、金山电池医生、别踩白块2等。这些拥有海量用户的产品也产生了海量的数据，针对这些产品和数据自然也会有实时性要求比较高的需求。

早期，我们在解决实时需求上并没有使用 Spark Streaming 或 Storm 等分布式计算引擎，而是使用了 RabbitMQ 和单机 Python 脚本的方式。然而，随着数据量的迅速增长和不同实时需求的提出，这种比较原始的方式慢慢暴露出了几个严重的缺陷：

- 单机处理能力有限。当数据量大了以后，单台机器的 CPU、内存、I/O、网络都很容易成为瓶颈。

- 可靠性差。单机出现意外，比如宕机、磁盘错误、网络故障等，所有的计算都将受到极大影响甚至灰飞烟灭。

- RabbitMQ 吞吐量较小。RabbitMQ 侧重于可靠性而非吞吐量，其并发性能难以支撑迅速增长的数据量。

基于以上缺陷，需要引入新的基于分布式、高可用、高吞吐量的实时计算架构。最终，我们选定了以 Spark Streaming+Kafka 为基础的方案，除了能解决上述几个严重问题外，以下几点也是我们的考量：

1. Kafka 支持数据重放，即可以多次消费同一份数据，这保证我们在代码 bug 及需求变动等情况下可以进行重算。

2. 团队对于 Spark 及 Kafka 均有使用经验，这能让我们更快更稳健地构建实时计算平台。

3. Spark 的高速发展及 Spark Streaming 对 Kafka 支持程度高。

基于上述原因，我们认为这样的架构能满足我们当前及未来较长一段时间的需求。

### 应用场景

基于 Spark Streaming 的实时计算在我们业务中主要有以下几个方面的应用：实时数据清洗；实时计算重要产品三维数据（三维指活跃、安装、卸载）；实时计算广告推广类各种指标数据；对于内、外部系统异常的实时监控报警；实时推荐及实时反作弊。

### 框架设计与实现

图1代表了当前实现中一个实时需求主要涉及的三大部分，即：输入相关（红色箭头）、计算相关（紫色箭头）、输出相关（蓝色箭头）。下面对这三部分进行介绍。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57283fcb6e837.jpg" alt="图1 一个streaming application及其三部分" title="图1 一个streaming application及其三部分" />

**图1 一个 streaming application 及其三部分**

### 输入相关

我们使用 Direct API 来创建 DStream，即 KafkaUtils#createDirectStream，这种方式，需要自己管理从哪里开始消费及消费到哪里了的 offsets，相对于使用 High Level Consumer API 的 KafkaUtils#createStream 来说比较麻烦，但是可以提高灵活性。这种能够自行控制 offsets 的灵活性对于业务来说非常必要，所以我们自己管理 offsets。我们需要的 offsets 共有两种：整点 offsets 和 last consumed offsets （即某 application 上一次消费某一个 topic 的 offsets，将在输出相关介绍）。

#### 整点的offsets

保留整点的 offsets 是为了减小重算的代价。虽然这里是实时计算平台，但有时候也会需要计算已经过去的时间的数据，比如：

- 产品早上8点要求改动某需求原计算逻辑，并要求当天全天的计算都按新的逻辑来算。那么，在9:15改完计算代码后，我们可以同时起两个  application：

  1. 从 Zookeeper 中取出0~9点的 offsets，计算这段 offsets 区间内的数据。

  2. 从 Zookeeper 中取出9点的 offsets，对每个 partition 的 offset 加1后作为 fromOffsets 来计算9点直至当前的数据。

- 代码bug导致之前算出的结果有误，修正代码后，需要重算之前的数据。且同时，要实时计算当前的数据。具体做法与上一个类似。

相较于没有整点 offsets，具有整点 offsets 能让我们在计算过去某段时间区间内的数据可以比较精确地定位到 offsets 区间而不用重新消费、计算所有的数据，并可以尽快实时处理当前的数据，大大减小了重算的代价。

#### 其他类型的 offsets

当然，如果你觉得整点的 offsets 还是不够精确，你可以保留每半小时的，每五分钟的，甚至每一分钟的。注意，这种方式会导致极少部分的数据没被处理或重复处理，这样极小的误差在大部分业务中是可以接受的。

另外，保留整点 offsets 的方法也很简单，另起一个脚本程序，在你需要的时候从指定的 topic 拉取数据并将此时的 offsets 写到 ZooKeeper 自定义的目录中。

### 计算相关

我们使用的 Spark 版本比较老，是 Spark 1.3.1（虽然后来 Spark 版本一直快速更新，但 Spark 1.3.1一直能满足我们的需求，且升级版本是一个风险比较高的动作，所以一直没有升级版本）。对于要结合较长时间历史数据做实时统计的需求，比如要统计某个广告位这一天的实时展示、点击次数和人数，虽然 updateStateByKey 操作能实现，但效率太低，并不能用于线上。这时我们需要一个 “缓存”来辅助统计。Redis 因其高性能、高稳定、丰富的数据类型及操作等特性而成为我们的选择。

实时业务中，五花八门的计算逻辑，大都需要借由 Redis 做记次和排重的操作。比如需求“实时统计某广告位当天展示、点击的次数和人数”，说白了就是记次和 UUID 排重操作，下面通过优化记次和排重来介绍我们改进计算的过程。

#### redis/ledis 优化过程

**粗暴使用 Redis**

最开始我们的使用方式是极其不合理的，对于每一条数据都需要单独处理操作 Redis，伪代码如下（因为 UUID 多达上千万，所以这里没有使用 Set 进行排重）。

```
val lines = KafkaUtils.createDirectStream(...)
lines.mapPartitions { partition => 
  partition.foreach { line =>
    // 统计次数
    redisClient.incr("次数")
 
    // 统计人数，排重操作。若新来的 uuid 不存在（即 incr 返回1），则 uuid 个数加1
    if ( redisClient.incr(uuid) == 1 ) {
      redisClient.incr( "人数" )
    }
  }
}.print()
```

若一个 batch 有10000条数据，那么这个 batch 内会对 Redis 读写40000次以上，这会造成两个严重问题：大量集中的读写请求会对 redis 造成巨大压力，Redis 很容易成为瓶颈；大量耗时网络请求会导致对一个 batch 的处理时间变长，甚至导致 job 积压。

**批量处理**

为了减少读写 Redis 的次数，我们不再像前文那样对 parition 的每一条数据单独处理，而是将一个 partiton 的数据进行批量处理。针对记次和排重分别做了优化。

- 记次优化：一个 parition 中，先本地统计各个 key 要 incr 的总次数，然后一次性执行一次 incrby。这可以大大减少记次带来的 Redis 读写次数，在我们的实际应用中，用于记次的 Redis 读写，减少了上百倍（incr 即加1，incrby (100)即加100）。

- 排重优化：将一个 partition 的数据先在本地排重后，再将原来逐个操作的方式改为 pipeline 操作，通过避免大量 RTT 开销来节省排重消耗的时间。这一优化在我们的应用中能将排重时间消耗降低5~10倍。
当然，在这里记次也是用的 pipeline 操作。

**引入 Ledis**

经过上一步的优化，操作 Redis 的性能得到了极大的提升。但随着需要处理的数据越来越多，Redis 占用的内存也越来越大，因为内存的高成本，我们引入了 LedisDB。

LedisDB 是一个高性能的 NoSQL 数据库，类似 Redis，支持 redis 绝大部分协议，可以直接使用 redis client 来操作 LedisDB，也就是说原先借助 Redis 开发的 streaming application 可以一行代码都不改无缝移植到 LedisDB 上。最重要的是，LedisDB 的数据是存储在廉价的磁盘而不是昂贵的内存。

以下为 LedisDB 在我们实际应用中几个常用操作上的性能数据，虽比不上 Redis，但处理能力依然非常强。若使用 SSD 作为 LedisDB 存储介质，性能还可以提高。

```
SET: 43478.26 requests per second
GET: 81234.77 requests per second
INCR: 27367.27 requests per second
LPUSH: 21838.83 requests per second
LPOP: 19477.99 requests per second
LPUSH (needed to benchmark LRANGE): 24539.88 requests per second
LRANGE_100 (first 100 elements): 10293.36 requests per second
LRANGE_300 (first 300 elements): 5235.05 requests per second
LRANGE_500 (first 450 elements): 3893.17 requests per second
LRANGE_600 (first 600 elements): 3008.24 requests per second
MSET (10 keys): 22914.76 requests per second
```

鉴于此，对于一些数据量不是特别大的需求，我们使用 LedisDB 来代替 Redis，从整体上看，节省了非常多的内存空间。

**优化 Redis HLLC**

对于一些数据量特别大的包含排重的需求，Ledis 的性能不足以应对，Redis 性能足够但却消耗大量内存，比如实时统计某产品活跃用户数，假设该产品当天有3000 w 活跃用户，每个 UUID 为32字节，那么 Redis 大约需要900 M 内存来支持这样一个统计。当有几十甚至上百这样的统计的时候，将占用非常多的内存空间。

为了减少类似需求的内存空间占用，我们使用并改进了 Redis HLLC （基数估算）。改进后的 HLLC 能让我们仅消耗512 KB 内存便可以支持数亿条数据的排重。当然，即使是改进了，HLLC 仍有1/10000~3/1000的精度损失。虽有一些精度损失，但对于大部分关注趋势和转化率的需求来说是完全可以接受的。

**引入 Redis Cluster**

最后，我们引入了 Redis Cluster，使其 QPS 和空间得到了近似线性扩展。

**mapWithState，脱离 Redis 进行统计的新机会**

虽然我们对借助 Redis/Leids 的操作进行了多轮优化，优化后也足以满足当前的需求。但是，从设计角度来说，这种方式始终增加了模块之间的耦合，所以我们也一直希望能脱离 Redis 进行要结合较长时间历史数据的实时计算。直到 Spark1.6推出了新特性 mapWithState，给了我们新的机会。抛开 mapWithState 的实现原理不谈，就使用来说，和 updateStateByKey 类似，可以实现结合较长时间历史数据的计算，更重要的是 mapWithState 的性能相比于 updateStateByKey 提升了10倍以上。

利用 mapWithState 的方案还在测试中，这里主要给和我们一样想脱离第三方“缓存” 的朋友提供一个思路。

#### 输出相关

输出部分主要涉及将每个 batch 统计后的结果写到 DB 或通过业务接口推送给业务系统，以及更新 last consumed offsets 至ZooKeeper。

这里主要介绍下第2点。last consumed offsets，即在每个 batch 结束前，会将本 batch 消费到的 offsets 更新到 ZooKeeper 中。这里 last consumed offsets 的作用是为了能让 application 因意外挂掉之后从上一次消费到的 offsets 处重启，这样能够保证“进度”不会丢失。

每个 batch 就更新一次 offset 会不会太频繁？其实不会，因为每次写入 ZooKeeper 的数据仅仅是各个 partiton 的 offset，数据量非常小，这点压力对 ZooKeeper 来说可以忽略。对比于 checkpoint 机制默认每个 batch duration 将类实例序列化存储到 HDFS/s3 来说数据量要更小且更快。

Spark Streaming checkpoint 机制也能达到保存 “进度”的效果，为什么我们不直接用 checkpoint 机制呢？原因有二：

- checkpoint 在 streaming application 改动并重新编译后不可用。checkpoint 文件是类 Checkpoint 的实例序列化的结果，若对代码改动后再编译，从 checkpoint 文件中恢复 StreamingContext 将因为无法反序列化而失败。所有“进度”都将丢失。

- 恢复不支持修改配置。checkpoint 中包含该 streaming application 的所有 conf，若 StreamingContext 能从 checkpoint 成功恢复，则没有方法能修改配置。比如，streaming app 运行了很长时间，因产生过多内存碎片而频繁 GC 导致每个 batch 耗时都超过 batch duration，这时你希望能停掉application，重启时能保留进度并给予设置更大的内存，checkpoint 机制是做不到的。