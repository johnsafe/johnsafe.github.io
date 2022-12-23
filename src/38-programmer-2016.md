## Spark 多数据源计算实践及其在 GrowingIO 的实践

文/田毅

本文主要介绍如何使用 Apache Spark 中的 DataSource API 以实现多个数据源混合计算的实践，那么这么做的意义何在，其主要归结于3个方面：

首先，我们身边存在大量的数据，结构化、非结构化，各种各样的数据结构、格局格式，这种数据的多样性本身即是大数据的特性之一，从而也决定了一种存储方式不可能通吃所有。因此，数据本身决定了多种数据源存在的必然性。
 
其次：从业务需求来看，因为每天会开发各种各样的应用系统，应用系统中所遇到的业务场景是互不相同的，各种各样的需求决定了目前市面上不可能有一种软件架构同时能够解决这么多种业务场景，所以在数据存储包括数据查询、计算这一块也不可能只有一种技术就能解决所有问题。

最后，从软件的发展来看，现在市面上出现了越来越多面对某一个细分领域的软件技术，比如像数据存储、查询搜索引擎，MPP 数据库，以及各种各样的查询引擎。这么多不同的软件中，每一个软件都相对擅长处理某一个领域的业务场景，只是涉及的领域大小不相同。因此，越来越多软件的产生也决定了我们所接受的数据会存储到越来越多不同的数据源。

### Apache Spark 的多数据源方案

传统方案中，实现多数据源通常有两种方案：冗余存储，一份业务数据有多个存储，或者内部互相引用；集中的计算，不同的数据使用不同存储，但是会在统一的地方集中计算，算的时候把这些数据从不同位置读取出来。下面一起讨论这两种解决方案中存在的问题：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad25e0c8d0.jpg" alt="图1  多数据源方案" title="图1  多数据源方案" />

图1  多数据源方案

第一种方案中存在的一个问题是数据一致性，一样的数据放在不同的存储里面或多或少会有格式上的不兼容，或者查询的差异，从而导致从不同位置查询的数据可能出现不一致。比如有两个报表相同的指标，但是因为是放在不同存储里查出来的结果对不上，这点非常致命。第二个问题是存储的成本，随着存储成本越来越低，这点倒是容易解决。

第二种方案也存在两个问题，其一是不同存储出来的数据类型不同，从而在计算时需求相互转换，因此如何转换至关重要。第二个问题是读取效率，需要高性能的数据抽取机制，尽量避免从远端读取不必要的数据，并且需要保证一定的并发性。

Spark 在1.2.0版本首次发布了一个新的 DataSourceAPI，这个 API 提供了非常灵活的方案，让 Spark 可以通过一个标准的接口访问各种外部数据源，目标是让 Spark 各个组件以非常方便的通过 SparkSQL 访问外部数据源。很显然，Spark 的 DataSourceAPI 其采用的是方案二，那么它是如何解决其中那个的问题的呢？

首先，数据类型转换，Spark 中定义了一个统一的数据类型标准，不同的数据源自己定义数据类型的转换方法，这样解决数据源之间相互类型转换的问题；

关于数据处理效率的问题，Spark 定义了一个比较简单的 API 的接口，主要有3个方式：

```
/* 全量数据抽取 */
trait TableScan {
def buildScan(): RDD[Row]
}
 
/* 列剪枝数据抽取 */
trait PrunedScan {
def buildScan(requiredColumns: Array[String]): RDD[Row]
}
 
/* 列剪枝＋行过滤数据抽取 */
trait PrunedFilteredScan {
def buildScan(requiredColumns: Array[String], filters: Array[Filter]): RDD[Row]
}
```

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad9a8593df.jpg" alt="图2  External Datasource API" title="图2  External Datasource API" />

图2  External Datasource API

1. TableScan。这种方式需要将 1TB 的数据从数据抽取，再把这些数据传到 Spark 中。在把这 1TB 的数据穿过网络 IO 传给 Spark 端之后，Spark 还要逐行的进行过滤，从而消耗大量的计算资源，这是目前最低效的方式。
2. PrunedScan。这个方式有一个好处是数据源只需要从磁盘读取 1TB 的数据，并只返回一些列的数据，Spark 不需要计算就可以使用 1GB 的数据，这个过程中节省了大量的网络 IO。 
3. PrunedFilteredScan。它需要数据源既支持列过滤也支持行过滤，其好处是在磁盘 IO 这一层进行数据过滤，因此如果需要 1GB 数据，可能只抽出 2GB 大小，经过列过滤的规则再抽出 1GB 的数据，随后传给 Spark，因此这种数据源接口最高效，这也是目前市面上实现的最高效的数据接口。

### 可直接使用的 DataSource 实现

目前市面上可以找到的 Spark DataSource 实现代码有三大类：Spark 自带；Spark Packages(http://Spark-packages.org/)网站中存放的第三方软件包；跟随其他项目一同发布的内置的 Spark 的实现。这里介绍其中几个：

#### JDBCRelation

```
private[sql] case class JDBCRelation(
url: String,
table: String,
parts: Array[Partition],
properties: Properties = new Properties())(@transient val sqlContext: SQLContext)
extends BaseRelation
with PrunedFilteredScan
with InsertableRelation {
....
}
```

以 JDBC 方式连接外部数据源在国内十分流行，Spark 也内置了最高效的 PrunedFilteredScan 接口，同时还实现了数据插入的接口，使用起来非常方便，可以方便地把数据库中的表用到 Spark。以 Postgres 为例：

```
sqlContext.read.jdbc(
“jdbc:postgresql://testhost:7531/testdb”,
“testTable”,
“idField”, -------索引列
10000, -------起始 index
1000000, -------结束 index
10, -------partition 数量
new Properties
).registerTempTable("testTable")
```

实现机制：默认使用单个 Task 从远端数据库读取数据，如果设定了 partitionColumn、lowerBound、upperBound、numPartitions 这4个参数，那么还可以控制 Spark 把针对这个数据源的访问任务进行拆分，得到 numPartitions 个任务，每个 Executor 收到任务之后会并发的去连接数据库的Server读取数据。

具体类型：PostgreSQL， MySQL。

问题：在实际使用中需要注意一个问题，所有的 Spark 都会并发连接一个 Server，并发过高时可能会对数据库造成较大的冲击（对于 MPP 等新型的关系型数据库还好）。

建议：个人感觉，JDBC 的数据源适合从 MPP 等分布式数据库中读取数据，对于传统意义上单机的数据库建议只处理一些相对较小的数据。

#### HadoopFsRelation

第二个在 Spark 内置的数据源实现，HadoopFs，也是实现中最高效的 PrunedFilteredScan 接口，使用起来相对来说比JDBC更方便。

```
1.sqlContext
2..read
3..parquet("hdfs://testFS/testPath")
4..registerTempTable("test")
```

实现机制：执行的时候 Spark 在 Driver 端会直接获取列表，根据文件的格式类型和压缩方式生成多个 TASK，再把这些 TASK 分配下去。Executor 端会根据文件列表访问，这种方式访问 HDFS 不会出现 IO 集中的地方，所以具备很好的扩展性，可以处理相当大规模的数据。

具体类型：ORC，Parquet，JSon。

问题：在实时场景下如果使用 HDFS 作为数据输出的数据源，在写数据就会产生非常大量零散的数据，在 HDFS 上积累大量的零碎文件，就会带来很大的压力，后续处理这些小文件的时候也非常头疼。

建议：这种方式适合离线数据处理程序输入和输出数据，还有一些数据处理 Pipeline 中的临时数据，数据量比较大，可以临时放在 HDFS。实时场景下不推荐使用 HDFS 作为数据输出。

#### ElasticSearch

越来越多的互联网公司开始使用 ELK（ElasticSearch＋LogStash＋Kibana）作为基础数据分析查询的工具，但是有多少人知道其实 ElasticSearch 也支持在 Spark 中挂载为一个 DataSource 进行查询呢？

```
1.EsSparkSQL
2..esDF(hc,indexName,esQuery)
3..registerTempTable(”testTable”)
```

实现机制：ES DataSource 的实现机制是通过对 esQuery 进行解析，将实际要发往多个 ES Nodes 的请求分为多个 Task，在每个 Executor 上并行执行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad7cf35f81.jpg" alt="图3  ElasticSearch架构" title="图3  ElasticSearch架构" />

图3  ElasticSearch 架构

问题：原生程序使用 HTTP 方式进行数据加载，吞吐量很低，需要修改为 Transport 方式。

建议：存储 doc 数据，随机数据搜索场景使用，做其他数据源的 Index。

#### Apache Phoenix（https://github.com/apache/phoenix）

Phoenix 提供了一个通过 SQL 方式访问 HBase 数据的途径，在不了解 HBase 实现细节的情况下很方便读写数据。其属于阿帕奇官方发布，位于 Apache Phoenix 项目里面的子模块。
 
实现机制：Phoenix 的实现机制是通过对 SQL 解析，将执行计划中并行的部分转换为多个 Task 在 Executor 上执行。

```
1.sqlContext
2..read 
3..format("org.apache.phoenix.Spark")
4..options(Map(“table” -> table, “zkUrl” -> zookeeperUrl))
5..load.registerTempTable(“testTable”)
```

问题：需要对 Phoenix 表模型非常了解，需要使用 Rowkey 字段进行查询。

建议：实时处理输出的数据，如果后面要进行数据查询，也可以把这个数据直接插入到 Apache Phoenix，这样后面查询的数据可以及时得到更新结果。

#### 其他

MongoDB－https://github.com/Stratio/Spark-mongodb 
Cassantra－https://github.com/datastax/Spark-cassandra-connector

### GrowingIO 实践

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad8af8b678.jpg" alt="图4  整体架构" title="图4  整体架构" />

图4  整体架构

GrowingIO 的数据平台主要分为两部分应用。首先是实时应用，负责实时处理流入数据的 ETL，将清洗后的数据录入 HBase 与 ES；然后是离线应用，负责定时执行离线模型计算，将实时数据的结果进行汇总分析。上层搭建了自己的 QueryService，负责根据前端需要快速查询返回需要的统计数据。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad8b9e9592.jpg" alt="图5  实时计算架构" title="图5  实时计算架构" />

图5  实时计算架构

实时计算部分最主要的功能是数据的 ETL，当实时数据从 Kafka 消费后，会利用 Spark 提供的 JSon DataSource 将数据转化为一个 Table，再通过 JDBC 将配置数据引入，通过 Phoenix 和 ES 的 DataSource 将最终存储的目标位置映射成为多个 Table，最终通过 SparkSQL 的 load 操作插入目标数据源。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad8c37b9e8.jpg" alt="图6  离线计算架构" title="图6  离线计算架构" />

图6  离线计算架构

离线部分的主要工作是二次汇总分析，这里将模型计算所需的内容从各个数据源挂载到 Spark，然后写很多复杂的 SQL 进行计算，再将结果保存到 HBase 供 QueryService 使用。

### 外部数据源使用中遇到的问题和解决途径

问题：Elastic Search 查询数据时，当 Mapping 数据的列大于 Source 中列时报 Index Out of Bound Exception。

解决：修改 RowValueReader 的 addToBuffer 方法。

问题：Elastic Search 数据加载默认通过 HTTP 的接口加载数据，性能极差。
 
解决：修改为 Transport 方式加载使得性能提升2-3倍。

问题：Elastic Search 性能优化。

解决：需要详细设计 Index, 尽量减少每次查询的数据量。

问题：Phoenix4.4 与 Spark 1.5兼容性。

解决：Spark 1.5修改的 DecimalType 类型适配，GenericMutableRow 修改为 InternalRow。

问题：PHOENIX-2279 Limit 与 Union All 相关的 BUG。

解决：修改 Phoenix 代码。

问题与解决：Phoenix 打包过程中解决与 Hadoop 版本兼容性。

问题与解决：Region Split 导致缓存中的 Region 信息失效（暂时无解）。

问题：由于 YARN 资源控制导致 Excutor 端报错：Phoenix JDBC Driver has been closed。

解决：配置额外的内存避免 Executor 被 YARN 杀掉。

问题：PhoenixRDD 读取数据时 Partition 数量过少导致读取速度慢。

解决：通过 Phoenix 的 BucketTable 增加查询的并行度（建议控制 Bucket 的数量，避免 Table 自动 split，BucketTable 在 split 后有 BUG）。

### 其他问题——Spark Job Server

由于大量使用了 SparkSQL 和 DataSource，所以面临到的一个新问题就是如何更加有效地去提升 Spark 平台的资源利用率。 社区中提供了一个开源的 Spark Job Server 实现（https://github.com/Spark-jobserver/Spark-jobserver） ，相较之下，觉得这个实现对于 GrowingIO 来说有些复杂,于是自己设计实现了一个简化版的 JobServer。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56aad978b37cb.jpg" alt="图7  JobServer架构" title="图7  JobServer架构" />

图7  JobServer 架构

简化版的 JobServer 允许用户通过一个 SDK 提交自己的 Job（需要提前部署对应的 jar 包到 JobServer）；JobServer 使用 FairScheduler 平均分配每个 Job 使用的资源；允许设置最大的 Job 并发数量；允许在 Job 中设置优先级别。

### 总结

总体来说，GrowingIO 通过使用 SparkSQL 加 DataSourceAPI 的方法在很短时间内搭建起一套完整的数据处理平台，并且扩展性很好。对于大多数中小企业来说，是一个便捷有效的途径。 

由于 SQL 的易学，对于团队中的新人上手也是比较容易。
 
但是大量使用 SQL 也带来一些问题，比如：所有数据模型的管理，例如所有字段的长度，类型的统一；需要根据性能监控对各个数据源进行不断的优化（例如 ES 的 Index，HBase 的 Rowkey）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56af272c6e818.jpg" alt="" title="" />