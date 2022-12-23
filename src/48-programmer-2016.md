## Streaming DataFrame：无限增长的表格

文/朱诗雄

在过去的一年中，Spark 项目高速发展，发布了多个重要版本和各种新特性，整个开源社区十分活跃，JIRA 数目和 PR 数目都突破了一万，参与贡献的人数也超过了一千，成为了最火的开源大数据项目。在2016年中，Spark 将继续保持飞速发展。今年的重头戏就是即将发布的
 Spark 2.0。在这个新版本中有许多值得期待的内容。本文将要介绍的是 Spark Streaming 的一个重量级特性：Streaming DataFrame （在笔者撰写本文时，Spark 2.0还未发布，很多内容还没有最后敲定，文中的某些内容可能会跟最后发布的版本有所出入）。

### 全新的数据模型

随着 DataFrame 的发展，它将逐渐代替原生 RDD，成为用户使用 Spark 的最主要 API。然而，目前的 Streaming DStream 模型依赖于 RDD，存在诸多不便。比如，Streaming 用户需要在批处理和流处理两个模型中进行切换，针对不同模型编写不同的代码；在使用
 DataFrame 时还需要在 RDD 和 DataFrame 之间进行转换。为了解决这些问题，Spark 2.0将推出 Streaming DataFrame，使用一致的数据模型，让用户理解起来更加容易，无须关心程序是批处理还是流处理，可以统一用 DataFrame 进行编程。

Streaming DataFrame 是 Spark 社区根据过去三年从 Spark Streaming 中学到的经验以及 Databricks 众多客户的反馈，基于 Spark 1.3引入的 DataFrame，从头构建和实现的 Streaming 引擎。

Streaming DataFrame 引入了一个全新的数据模型 Repeated Queries。在新的模型中，Stream 的数据源从逻辑上来说是一个
 Append-Only 的动态表格，随着时间的推移，新数据被持续不断地添加到表格的末尾。用户可以使用 DataFrame 或者 SQL 来对这个动态的输入数据源进行实时查询。每次查询在逻辑上就是基于当前的表格内容执行一次 SQL 查询。Stream 的查询如何执行则是由用户通过触发器（Trigger）来设定。用户既可以设定定期执行，也可以让查询尽可能快地执行，从而达到实时的效果。Stream 的输出有多种模式，既可以是基于整个 Stream 输入执行查询后的完整结果，也可以选择只输出与上次查询相比的差异，或者就是简单地追加最新的结果。

这个模型对于熟悉 SQL 的用户来说，将不用再学习新的概念。对流的查询跟查询一个表格几乎完全一样。

### 流动的 DataFrame

基于上述模型，Spark 2.0中将会引入一个全新打造的执行引擎来支持 Streaming DataFrame，原本用于操作表格的 DataFrame 将可以直接用于操作 Stream。也就是说，批处理和流处理可以统一使用 DataFrame 来进行操作。对于批处理来说，DataFrame 是长度有限的表格，对于流处理来说，DataFrame 则是无限增长的表格（如图1）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1972c2262.png" alt="图1  统一的抽象DataFrame" title="图1  统一的抽象DataFrame" />

**图1 统一的抽象 DataFrame**

将 DataFrame 作为统一的抽象，带来了许多优点。

- 用户无须考虑批处理和流处理，可以使用一致的模型进行思考。

- 一致的 API 让用户可以更加便捷地搭建一体化的大数据流水线，实现 Lambda 架构。

- Streaming DataFrame 可以享受 SQL 查询优化器（Catalyst）和钨丝计划（Project Tungsten）带来的各种性能优化。

- 支持多个 Stream 同时独立运行，也允许用户动态地修改查询。

- Streaming DataFrame 可以支持交互式的查询。

### 一致的 API

- 在 API 层面，DataFrame 作为 SQL 和 Streaming 的统一抽象，用户既可以使用传统的 SQL 数据源来创建 DataFrame，也可以使用 DataFrameReader 新增的 stream 方法来接入新的 Stream 数据源。而 Stream 的输出则可以使用 DataFrameWriter 新增的 startStream 方法来设置。

```
logs = ctx.read.format("json").open("hdfs://logs")
logs.groupBy(logs.user_id).agg(sum(logs.time))
       .write.format("jdbc").save("jdbc:mysql//...")
```

**代码1  传统 DataFrame 示例**

```
logs = ctx.read.format("json").stream("hdfs://logs")
logs.groupBy(logs.user_id).agg(sum(logs.time))
       .write.format("jdbc").startStream("jdbc:mysql//...")
```

**代码2  Streaming DataFrame 示例**

代码1使用了 DataFrame 对用户日志进行分析统计，计算不同用户的在线时间。代码2则是使用新的 Streaming DataFrame 来实时输出不同用户的在线时间。可以看到，除了输入和输出使用的方法不同，两段代码几乎完全一样。此外，用户还可以使用 startStream 返回的 ContinuousQuery 来查询和管理运行中的 Query。

### 写在最后

为了实现这个模型，社区攻克了许多难题。比如，需要精确定义基于这个模型的各种操作的语义；如何用有限的资源来支持无限增长的表格等。在即将发布的 Spark 2.0中，Streaming DataFrame 将会有一个初步的版本，本文提到的功能已经基本实现，同时 Stream 的 Aggregation 操作也正在火热开发中。对实现细节感兴趣的读者可以关注 SPARK-8360。

相信 Spark 2.0的发布将会把 Spark 带上一个新的台阶，让大数据处理和分析更加简单。同时也期待看到更多国内的工程师参与到 Spark 社区中。