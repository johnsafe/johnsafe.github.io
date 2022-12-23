## 百度分布式交互查询平台——PINGO 架构迭代

文/温翔，沈光昊，蔡旻谐，徐宝强，刘少山

PINGO 是由百度大数据部与美国研发中心合作开发的分布式交换查询平台。在它之前，百度的大数据查询作业主要由基于 Hive 的 QueryEngine 完成。QueryEngine 很好地支持着离线计算任务，但对交互式的在线计算任务支持并不好。为此，在一年前设计了基于 SparkSQL 与 Tachyon 的 PINGO 的雏形。他们在过去一年中，通过跟不同业务的结合，PINGO 逐渐的演变成一套成熟高效的交互式查询系统。本文将详细介绍其的架构迭代过程以及性能评估。

### PINGO设计目标
<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5302040856.png" alt="图1 Query Engine的执行流程" title="图1 Query Engine的执行流程" />

图1 Query Engine 的执行流程

QueryEngine 是基于 Hive 的百度内部大数据查询平台，这套系统在过去几年较好支撑了相关业务。图1展示了 QueryEngine 的架构，其服务入口叫做 Magi。用户向 Magi 提交查询请求，Magi 为这次请求，分配一个执行机，执行机会调用 Hive 读取 Meta 信息并向 Hadoop 队列提交任务。在这一过程中，用户需要自行提供计算所需的队列资源。随着近几年对大数据平台的要求越来越高，我们在使用 QueryEngine 过程中也发现了一些问题：首先 QueryEngine 需要由用户提供计算资源，这使得数据仓库用户需要了解 Hadoop 以及相关的管理流程，增加了用户使用数据的门槛。 第二，对于很多小型计算任务而言，MR 的任务的起动时间也较长，往往计算任务只需要1分钟，但是排队/提交任务就需要超过1分钟甚至更长的时间。结果是，QueryEngine 虽然很好地支持线下执行时间长的任务，但是对线上的一些交换式查询任务（要求延时在一到两分钟内）确是无能为力。

为了解决这些问题，大约一年前，我们尝试在离线计算的技术栈上搭建起一套具有在线服务属性的 SQL 计算服务 PINGO。如图2所示：PINGO 使用了 SparkSQL 为主要的执行引擎，主要是因为 Spark 具有下面的特点。
<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5302f4ce45.png" alt="图2 PINGO初版系统架构设计" title="图2 PINGO初版系统架构设计" />

图2 PINGO初版系统架构设计

内存计算。
Spark 以 RDD 形式把许多数据存放在内存中，尽量减少落盘，从而提升了计算性能。

可常驻服务。
Spark 可以帮助实现常驻式的计算服务，而传统的 Hadoop 做不到这一点。常驻式的计算服务有助于降低数据查询服务的响应延迟。
机器学习支持。
对于数据仓库的使用，不应仅仅局限于 SQL 类的统计任务。Spark 的机器学习库可以帮助我们将来扩展数据仓库，提供的交互式的数据挖掘功能。

计算功能多元。
虽然 PINGO 是一个查询服务，不过仍然有其他类型的计算需求，如数据预处理等。 使用 Spark 可以使我们用一个计算引擎完成所有的任务，简化系统的设计。
### PINGO 系统迭代
在过去一年中，PINGO 从雏形开始，通过跟不同业务结合，逐渐的演变成一套成熟高效的系统。中间经历过几次的架构演变，我们将详细介绍其的迭代过程。
#### PINGO 1.0
PINGO 初版的目标是提升性能，令其能支持交互式查询任务。由于 Spark 是基于内存的计算引擎，相对 Hive 有一定性能优势，所以第一步我们选择使用 Spark SQL。为求简单，最初的服务是搭建在 Spark Standalone 集群上的。我们知道，Spark 在 Standalone 模式下是不支持资源伸缩的， 一个 Spark Application 在启动时会根据配置获取计算资源。这时就算查询请求只有一个 Task 还在运行，该 Application 所占用的所有资源都不能释放。好在一个 Spark Application 可以同时运行多个 Job，每个 Job 都能够“平分”其所占用的计算资源。

基于上面的考虑，如图3所示，我们利用 Spark 的 Standalone 集群搭建了第一版 PINGO 服务，我们的主服务节点叫做 Operation Manager，它本身也是一个 Spark Driver。所有用户的请求都会从 Magi Service 发送到 Magi Worker，再由 Magi Worker 分配给 Operation Manager，通过 Operation Manager 在 Spark 集群上执行。
 
<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5303c3bb86.png" alt="图3 PINGO 1.0 系统架构" title="图3 PINGO 1.0 系统架构" />

图3 PINGO 1.0 系统架构
#### PINGO 1.1
通过 PINGO 1.0，我们把数据仓库的计算引擎从 Hive/MR 转变成了 Spark。不过，单纯替换计算引擎，也许可以把查询性能提升1-2倍，但并不会使查询的速度有数量级的提高，想要从本质上提高查询速度，我们还需要改进存储。对此我们设计了 PINGO 1.1，加入了以 Tachyon 为载体的缓存系统，并在缓存层上搭建了缓存管理系统 ViewManager，进一步提升系统性能。 

很多快速的查询系统都将 Parquet 之类列存储格式和分布式 KeyValue 存储引擎结合起来，通过建立索引/物化视图等手段加速 SQL 查询的速度。通过将部分查询条件下推到存储引擎，某些 SQL 查询语句甚至可以被提速至100倍以上。然而我们的数据都存储在旧版 Hive 数据仓库中，文件系统被限定为 HDFS，数据格式一般为低版本的 ORC File，也有不少表是 ORCFile 或者文本。因为迁移成本/数据上下游依赖等兼容性等原因，我们没有办法更改 Hive 数据仓库本身的存储。

不过根据局部性原理，任何数据访问都有热点. 我们可以建立缓存系统，在其中应用最新的文件系统和存储格式. 通过把热点输入通过预加载的方式导入到缓存系统，我们可以加速整个系统的访问性能.为此，我们设计了以下的系统架构。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53047a45cb.png" alt="图4 PINGO 1.1 系统架构" title="图4 PINGO 1.1 系统架构" />

图4 PINGO 1.1 系统架构

架构中引入了一个新模块 ViewManager，负责管理缓存系统。它的主要功能是识别热点数据，将导入到缓存系统，并在查询时改变 SQL 的执行过程，使得 Spark 从缓存而不是原始位置读取数据。在这个架构下，当一个 Query 进来时，会先向 OperationManager 请求。当接受到请求后，OperationManager 会向 ViewManager 查询数据是否已经在缓存中。如果已经在缓存，数据将被直接返回给用户， Query 完成。如果不在，OperationManager 会向底层 Data Warehouse 发起请求，等数据到达时返回用户。同时，ViewManager 也会向底层 Data Warehouse 发起请求，等数据到达时建立一个 Cache Entry， 这样下次同样 Query 进来时，就会从缓存读数据。注意这里我们 OperationManager 与 ViewManager 向底层 Data Warehouse 的数据读取是两个独立 STREAM， 保证两者不会互相干扰。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5305a9b1fe.png" alt="图5 PINGO1.1 对SparkSQL的改进" title="图5 PINGO1.1 对SparkSQL的改进" />

图5 PINGO1.1 对 SparkSQL 的改进

Spark 中利用 Catalyst 框架来执行 SQL 语句。对于 Catalyst 框架来说，其输入是 Unresolved Logical Plan，可以简单理解为 SQL 语句的结构化描述。 Catalyst 调用 Catalog 来向 MetaService 查询表的元数据， Catalog 返回 MetastoreRelation 来代表 Hive 表，其中含有读取该表的所有必要信息，以便后续的解析和优化。而在执行时，Catalyst 会把 MetastoreRelation 转化为 HiveTableScan，来完成对数据的读取。

为了实现对缓存的透明利用，我们在其中两个地方了做了扩展。首先在 Catalog 中为缓存的表返回 CachableRelation 来代替 MetastoreRelation。而在将 LogicalPlan 转变为真正执行的 PhysicalPlan 时， 把 CachableRelation 翻译为两种 TableScan 的 Union，一个针对那些被缓存的数据，另外一个针对些没有被缓存的数据。

通过这种方式，我们能做到在用户不感知的情况下，完成对数据仓库存储层的替换和优化。目前我们做的仅仅是改变存储的文件系统和格式，将来也会将同样的实现思路应用到索引和物化视图上。
#### PINGO 1.2
PINGO 1.1服务提高了系统性能，但在运营了一段时间后，我们逐渐感觉 Spark 集群管理问题正成为整个系统的瓶颈，具体表现在两方面。

整个服务其实是一个 Spark Application，服务的主节点同时是 Spark 的 Driver。而我们知道， Spark 并不以高可靠性见长，我们在实践中也发现把所有的计算压力放到单个 Spark Application 容易导致比较严重的 GC 和负载问题。而当 Spark 出问题需要重启时，整个服务也就暂停了。

单一 Spark 集群的架构无法适应多机房的基础设施。百度内部的离线集群规模还是很大的，机房分布在全国多个地区。这样，我们的服务能够获取的资源往往来自多个机房，而多个机房的机器不适合组成一个 Spark 集群。另外，当集群读取其他机房的数据时，带宽往往成为整个计算任务的瓶颈。

发现这些问题是好事，说明系统已经逐渐成熟，开始往怎么更好的运维的方向发展了。 为了解决上面的问题，我们对系统进行了进一步的迭代. 迭代后的架构如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5306d6cec2.png" alt="图6 PINGO 1.2 系统架构" title="图6 PINGO 1.2 系统架构" />

图6 PINGO 1.2 系统架构

在这一版架构中，我们将 PINGO 的服务和调度 功能独立出来，与真正执行计算的部分剥离。 支持了单一查询入口 PinoMaster 进行调度，多个集群 Pingo Applicatoin 执行计算的多集群功能。 PingoMaster 同时维护多个 Spark Application。当一个 Query 到来时，我们会根据集群的归属/存储的位置选择一个最优的 Application 执行任务。 另外，这一版本的 PINGO 还支持在 YARN 集群上起动 Application。百度内部有自己的资源管理系统， 提供了和 YARN 兼容的接口。通过迁移 PINGO 服务到 YARN 上避免了原本 Standalone 版本需要的很多运维工作，并且可以通过定期重启 Application 来解决 Spark 的可靠性问题。

为了能够在 PINGO 中实现更好的调度策略，我们也对 Spark SQL 进行了深度扩展。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5307aa9d71.png" alt="图7 PINGO1.2对SparkSQL的改进" title="图7 PINGO1.2对SparkSQL的改进" />

图7 PINGO1.2对 SparkSQ 的改进

当 Pingo 收到一个 Query 后，我们在 Master 端就完成对这条 Query 的分析和部分优化，并将这些信息保存到 QueryContext 中。 当 SQL 在 Application 端真正执行时，我们从 Master 端而不是 Meta 端拿到执行所需要的信息。这样做的好处是可以根据 Query 的内容来做出调度。基于这个执行流程，我们开发了两个比较有效的调度策略:

1.根据数据的存储位置进行调度。因为我们在 Master 端就能知道 Query 所需数据的存储位置，所以可以选择合适的 PingoApplication 来执行这条 Query。

2.根据数据量大小进行调度。如上文所说，一个 Spark Aplication 可支持同时运行多个 Job，但很多大型的 Job 同时运行在一个 Application 又会造成 FullGC 等稳定性问题。我们会根据 Query 输入量的大小，为大型 Query 单独启动 Application，而让小 Query 复用同一个 Application。这样的好处是对于多数小 Query ，我们节省了起动 Application 的开销，而又避免了大 Query 造成的稳定性问题。
### PINGO 性能评估
<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5308c10dcf.png" alt="图8 Query执行时间: Hive vs. Spark" title="图8 Query执行时间: Hive vs. Spark" />

图8 Query 执行时间: Hive vs. Spark

前文详细介绍了 PINGO 架构的迭代，接下来，我们重点看PINGO的性能。如图8所示，首先我们对比了使用 Hive 以及使用 Spark 作为计算引擎的性能。 这里的 Benchmark 选取的 Query 是百度内部交互式数据分析的典型负载，主要是 Join/Aggregate 等操作，输入的数据量从100M 到2T 左右.我们可以看到，Spark 相比 Hive 有较大的性能优势。在比较大比较复杂的 Query 中， Spark 取得了2到3倍的加速比。注意在这个对比中，内存资源有限，为64GB，很多 Spark 数据被迫落盘，如果有更多内存，加速比会更高。

下一步我们了解加了缓存后的性能，在这个实验中，我们使用了192GB的机器，有更多内存支持缓存以及内存计算。如图9所示，在这个环境下，使用 Spark 相对于 Hive 有大概5到7倍的提速。加入了 Tachyon 后，相对于 Hive 无缓存的情况有30到50倍的提速。我们相信在未来的几年，内存价格会大幅降低，内存计算会变成主流，所以使用内存做缓存是一个正确的方向。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5309f7a970.png" alt="图9 缓存性能提高" title="图9 缓存性能提高" />

图9 缓存性能提高

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d530aaccb41.png" alt="图10 Query执行时间: Hive vs. PINGO" title="图10 Query执行时间: Hive vs. PINGO" />

图10 Query 执行时间: Hive vs. PINGO

PINGO 服务目前被应用在交互式查询场景中，旨在为 PM 和 RD 提供快速的数据分析服务。 图10展示了在同样的计算资源下，在生产环境使用 PINGO 前后的 Query 执行时间分布图。注意，在这个生产环境中，我们用的是64GB 内存机器， 为了提供足够的缓存空间，使用了 Tachyon 的 Tiered Storage 功能，让缓存分布在内存以及本地磁盘中。 我们可以看到，在传统的基于 Hive+MR 的服务模式下，只有1%左右的 Query 能在两分钟内完成. 而采用了基于 Spark 的 PINGO 服务，有50%以上的 Query 可以在两分钟内执行完成。 能够取得这样的加速效果，部分原因是 Spark 本身的执行效率比 Hive 要高一些。而且还有很大的优化空间，如果使用内存缓存，执行时间可以轻易压缩到30秒内。
### 结论和展望
经过一年的迭代与发展，PINGO 已很好支持了百度内部的交互式查询业务。PINGO，很多查询的延时从以前的30到40分钟，降低到了2分钟以内，大提高了查询者的工作效率。今后 PINGO 将朝着更好用，已及“更快”两个方向发展。为了让 PINGO“更好用”，我们正在 PINGO 之上开发 UI， 让用户与图形界面交互（而不是通过输入 SQL Query）就能轻易查询到想要的数据。为了更快，我们希望可以把90%以上的 Query 时间降低到30秒以下。第一步就是要扩大 Cache 的覆盖率与性能，设计出更好的缓存预取以及缓存替换机制，并且加速 Cache 读取的延时。第二步，我们也通过硬件加速的方法使用 FPGA 加速某些常用的 SQL Operator。就在最近，我们对 SQL 的 JOIN Operator 进行了 FPGA 硬件加速，相对 CPU 取得了3倍的加速比。我们也会很快把这项技术使用到 PINGO 上。