## Spark Streaming + ES 构建美团 App 异常监控平台

文/秦思源，王彬

>实时监控分析 App 异常，是业界流行的保证 App 质量的方法。但面对海量的异常数据流，普通的监控系统很难满足实时监控分析的需求。为了解决这个问题，我们结合了目前业界广泛应用的开源流式处理引擎  Spark Streaming 和搜索引擎 Elasticsearch，构建了一个低成本高可用的异常监控平台。本文将分享 Spark Streaming + ES 在美团 App 异常监控平台中的实践。

如果使用 App 时遇到闪退，你可能会选择卸载 App、到应用商店怒斥开发者等方式来表达不满。但 App 开发者也同样感到头疼，因为崩溃可能意味着用户流失、营收下滑。

为了降低崩溃率，提升 App 质量，开发团队需要实时监控异常。一旦发现严重问题，及时进行热修复，从而把损失降到最低。App 异常监控平台，就是将这个方法服务化。

本篇以核心需求为中心，逐一展开介绍如何使用 Spark Streaming + ES 构建 App 异常监控平台。

### 低成本

小型创业团队一般会选择第三方平台提供的异常监控服务。但中型以上规模的团队，往往因为不想把核心数据共享给第三方平台，而选择独立开发。

造轮子，首先要考虑的就是成本问题。我们选择了站在开源巨人的肩膀上，如图 1 所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f862e9023f5.png" alt="图1 数据流向示意图" title="图1 数据流向示意图" />

**图1 数据流向示意**

- Spark Streaming

每天来自客户端和服务器的大量异常信息，会源源不断地上报到异常平台的 Kafka 中，因此我们面临的是一个大规模流式数据处理问题。
美团数据平台提供了 Strom 和 Spark Streaming 两种流式计算解决方案。我们主要考虑到团队之前在 Spark 批处理方面有较多积累，使用 Spark Streaming 成本较低，就选择了后者。

- Elasticsearch

Elasticsearch（后文简称 ES）。是一个开源搜索引擎。不过在监控平台中，我们当做“数据库”来使用。为了降低展示层的接入成本，我们还使用了另一个开源项目 ES SQL 提供类 SQL 查询。

ES的运维成本，相对 SQL on HBase 方案也要低很多。整个项目开发只用了不到700行代码，开发维护成本还是非常低的。

如此“简单”的系统，可用性能保证吗？

### 高可用

Spark Streaming + Kafka 组合，提供了“Exactly Once”保证：异常数据经过流式处理保证结果数据中（注：并不能保证处理过程中），每条异常最多出现一次，且最少出现一次。

保证 Exactly Once 是实现24×7高可用服务最困难的地方。

在实际生产中会出现很多情况，对 Exactly Once 的保证提出挑战：

- 异常重启

Spark 提供了 Checkpoint 功能，可以让程序再次启动时，从上次异常退出的位置，重新开始计算。这就保证了即使发生异常，也可以实现每条数据至少写一次 HDFS。再覆写相同的 HDFS 文件就保证了 Exactly Once （注：并不是所有业务场景都允许覆写）。

写 ES 的结果也一样可以保证 Exactly Once。你可以把 ES 的索引，就当成 HDFS 文件一样来用：新建、删除、移动、覆写。

作为一个24×7运行的程序，在实际生产中，异常是很常见的，需要有这样的容错机制。但是否遇到所有异常，都要立刻挂掉再重启呢？

显然不是，甚至在一些场景下，即使重启了，还是会继续挂掉。我们的解决思路是：尽可能把异常包住，异常发生时，暂时不影响服务。

如图2所示，包住异常，并不意味可以忽略它，必须把异常收集到 Spark Driver 端，接入监控（报警）系统，人工判断问题的严重性，确定修复的优先级。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f8673f246dc.png" alt="图2  异常重启架构图" title="图2  异常重启架构图" />

**图2 异常重启架构图**

为了更好地掌控 Spark Streaming 服务状态，我们还单独开发了一个作业调度（重启）工具。

美团数据平台安全认证的有效期是7天，一般离线的批处理作业很少会运行超过这个时间，但 Spark Streaming 作业就不同了，它需要一直保持运行，所以作业只要超过7天就会出现异常。

因为没有找到优雅的解决方案，只好粗暴地利用调度工具，每周重启刷新安全认证，来保证服务的稳定。

- 升级重导

Spark 提供了2种读取 Kafka 的模式：Receiver-based Approach 和 Direct Approach。

使用 Receiver 模式，在极端情况下会出现 Receiver OOM 问题。
使用 Direct 模式可以避免这个问题。我们使用的就是这种 Low-level 模式，但在一些情况下需要我们自己维护 Kafka Offset：

**升级代码**：开启 Checkpoint 后，如果想改动代码，需要清空之前的 Checkpoint 目录后再启动，否则改动可能不会生效。但这样做了之后，就会发现另一个问题——程序“忘记”上次读到了哪个位置，因为存储在 Checkpoint 中的 Offset 信息也一同被清空了。这种情况下，需要自己用 ZooKeeper 维护 Kafka 的 Offset。

**重导数据**：重导数据的场景也如此，当希望从之前的某一个时间点重新开始计算时，显然也需要自己维护时间和 Offset 的映射关系。

自己维护 Offset 的成本并不高，所以看起来 Checkpoint 功能很鸡肋。其实可以有一些特殊用法的，例如，因为 Python 不需要编译，所以如果使用 Pyspark，可以把主要业务逻辑写在提交脚本的外边，再使用 Import 调用。这样升级主要业务逻辑代码时，只要重启程序即可。网上有不少团队分享过升级代码的“黑科技”，这里不再展开。
实现24×7监控服务，我们不仅要解决纯稳定性问题，还要解决延迟问题。

### 低延迟

App 异常监控，需要保证数据延迟在分钟级。虽然 Spark Streaming 有着强大的分布式计算能力，但要满足用户角度的低延迟，可不是单纯的计算完成这么简单。

- 输入问题

iOS App 崩溃时，会生成 Crash Log，但其内容是一堆十六进制的内存地址，对开发者来说就是“天书”。只有经过“符号化”开发者才能看懂。

我们将数据源分为符号化数据流、未符号化数据流，可以看出它们的相对延迟时间。

因为符号化需要在 Mac 环境下进行，而我们的 Mac 集群资源有限，不能符号化全部 Crash Log。即使做了去重等优化，符号化后的数据流还是有延迟。每条异常信息中，包含 N 维数据，如果不做符号化只能拿到其中的 M 维。

如图3所示，可以看出两个数据流的相对延迟时间 T 较稳定。如果直接使用符号化后的数据流，那么全部 N 维数据都会延迟时间 T。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f867ed7bf14.png" alt="图3  双数据流融合示意图" title="图3  双数据流融合示意图" />

**图3  双数据流融合示意图**

为了降低用户角度的延迟，我们根据经验加大了时间窗口：先存储未符号化的 M 维数据，等拿到对应的符号化数据后，再覆写全部 N 维数据，这样就只有 N - M 维数据延迟时间T了。

- 输出问题

如果 Spark Streaming 计算结果只是写入 HDFS，很难遇到什么性能问题。但如果想写入 ES，问题就来了。因为 ES 的写入速度大概是每秒1万行，只靠增加 Spark Streaming 的计算能力，很难突破这个瓶颈。

异常数据源的特点是数据量的波峰波谷相差巨大。由于我们使用了 Direct 模式，不会因为数据量暴涨而挂掉，但这样的“稳定”从用户角度看没有任何意义：短时间内，数据延迟会越来越大，暴增后新出现的异常无法及时报出来。

为了解决这个问题，我们制定了一套服务降级方案。

如图4所示，我们根据写 ES 的实际瓶颈 K，对每个周期处理的全部数据 N 使用水塘抽样（比例 K/N），保证始终不超过瓶颈。并在空闲时刻使用 Spark 批处理，将 N - K 部分从 HDFS 补写到 ES。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f868290c102.png" alt="图4  服务降级方案示意图" title="图4  服务降级方案示意图" />

**图4 服务降级方案示意图**

既然写 ES 这么慢，那我们为什么还要用 ES 呢？

### 高性能

App 开发者需要在监控平台上分析异常。

实际分析场景可以抽象描述为“实时、秒级、明细 、聚合” 数据查询。

我们使用的 OLAP 解决方案可以分为4种，它们各有优势：

- SQL on HBase 方案，例如：Phoenix、Kylin。我们团队从2015年Q1开始，陆续在 SEM、SEO 生产环境中使用 Phoenix、Kylin。Phoenix 算是一个“全能选手”，但更适合业务模式较固定的场景；Kylin 是一个很不错的OLAP产品，但它的问题是不能很好支持实时和明细查询，因为它需要离线预聚合。另外，基于其他 NoSQL 的方案，基本大同小异，如果选择 HBase，建议团队在其运维方面有一定积累。

- SQL on HDFS 方案，例如 Presto、Spark SQL。这两个产品，因为只能做到亚秒级查询，我们平时多用在数据挖掘场景中。

- 时序数据库方案，例如 Druid、OpenTSDB。OpenTSDB 是我们旧版
 App 异常监控系统使用过的方案，更适合做系统指标监控。

- 搜索引擎方案，代表项目为 Elasticsearch。相对上面的3种方案，基于倒排索引的 ES 非常适合异常分析的场景，可以满足：实时、秒级、明细、聚合，全部4种需求。

ES 在实际使用中的表现如何呢？

#### 明细查询

支持明细查询，算是 ES 的主要特色，但因为是基于倒排索引的，结果最多只能取到10000条。

在异常分析中，使用明细查询的场景，其实就是追查异常 Case，根据条件返回前100条就能满足需求了。例如：已知某设备出现了 Crash，直接搜索它的 DeviceId 就可以看到最近的异常数据。

我们在生产环境中做到了95%的明细查询场景1秒内返回。

#### 聚合查询

面对爆炸的异常信息，一味追求全是不现实，也是没必要的。App 开发者，需要能快速发现关键问题。

因此平台需要支持多维度聚合查询，例如按模块、版本、机型、城市等分类聚合，如图 5 所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f868baafaaf.png" alt="图5  聚合查询页面截图" title="图5  聚合查询页面截图" />

**图5 聚合查询页面截图**

不用做优化，ES 聚合查询的性能就已经可以满足需求。因此，我们只做了一些小的使用改进，例如很多异常数据在各个维度的值都是相同的，做预聚合可以提高一些场景下的查询速度。开发者更关心最近48小时发生的异常，分离冷热数据，自动清理历史数据也有助于提升性能。最终在生产环境中，做到了90%的聚合查询场景1秒内返回。

#### 可扩展

异常平台不止要监控 App Crash，还要监控服务端的异常、性能等。不同业务的数据维度是不同的，相同业务的数据维度也会不断变化，如果每次新增业务或维度都需要修改代码，整套系统的升级维护成本就会很高。

#### 维度

为了增强平台的可扩展性，我们做了全平台联动的动态维度扩展：如果 App 开发人员在日志中新增了一个“城市”维度，那么他不需要联系监控平台做项目排期，立刻就可以在平台中查询“城市”维度的聚合数据。

只需要制定好数据收集、处理、展示之间的交互协议，做到动态维度扩展就很轻松了。

需要注意的是，ES 中需要聚合的维度，Index 要设置为“not\_analyzed”。

想要支持动态字段扩展，还要使用动态模板，样例如代码 1所示：

```
{"mappings":
{"es_type_name":
  {"dynamic_templates":
   [ { "template_1":
       {"match": "*log*",
        "match_mapping_ type": "string",
        "mapping": {"type": "string" }
       }},
     {"template_2":
        { "match": "*",
          "match_mapping_ type": "string",
          "mapping": {"type": "string",
                              "index": "not_analyzed" } } }
   ] } } }
```

**代码1 ES 索引配置**

#### 资源

美团的数据平台提供了 Kafka（0.8.2.0）、Spark（1.5.2）、ES（2.1.1）的集群，整套技术栈在资源上也是分布式可扩展的。