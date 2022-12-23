## Spark 学习指南

文/周志湖

Spark 是当前最流行的开源大数据内存计算框架，采用 Scala 语言实现，由 UC 伯克利大学 AMPLab 实验室开发（2009）并于2010年开源，在2014年成为 Apache 基金会的顶级项目。2014年至2015年，Spark 经历了高速发展，Databricks 2015 Spark 调查报告显示：2014年9月至2015年9月，已经有超过600个 Spark 源码贡献者，而在此之前的12个月人数只有315，Spark 超越 Hadoop，无可争议地成为大数据领域内最活跃的开源项目。除此之外，已经有超过200个公司为 Spark 奉献过源代码，使 Spark 社区成为迄今为止开发人员参与最多的社区。

Spark 虽然也可以使用 Python、R、Java 语言进行多语言编程，但要想深入地理解 Spark 整体架构、运行原理及实现细节，Scala 语言的学习必不可少。Spark 的学习应该循序渐进，先学习语言的核心基本语法如函数式编程、面向对象编程、模式匹配、类型系统、隐式转换及并发编程模型等，在此基础上学习 Spark 的编程模型如常用 RDD 的类型、操作及计算原理等，再学习 Spark 的五大核心组件以对 Spark 中的交互式数据查询、图计算、流式计算及机器学习等有较深入的理解。具备一定基础后，通过对 Spark 内核源码如任务调度、底层资源管理、Shuffle 等进行分析，掌握 Spark 的内核原理，在此基础上分析 Spark SQL、Spark 流计算、 Spark 机器学习及 Spark 图计算等核心组件的源码实现，从而掌握各大组件的设计理念与思想。除此之外，还应该对 Spark 生态圈中重要的技术框架如 Docker、Kafka 及
 Alluxio 等，Spark 的性能调优等重要内容进行学习，从而做到真正掌握 Spark 核心技术。

### 六大分类

- Spark 入门

  掌握Scala语言编程基础，了解 Spark 的基本原理，在此基础上学习 Spark 开发环境部署、Spark 编程模型，掌握 RDD 的使用及运行原理，能够搭建 Spark 应用程序开发环境并学习应用程序的调试方法，理解 Spark 工作机制。

- Spark 核心组件

  包括 Spark SQL、Spark Graphx、Spark Streaming、Spark MLLib/ML、Spark R 五大核心组件的原理与应用。

- Spark 源码分析

  包括 Spark 内核源实现、Spark MLlib/ML 相关源码实现、Spark Streaming 相关源码实现、Spark Graphx 源码实现及 Spark SQL 源码实现。

- Spark 调优

  包括但不限于 Shuffle 模块的性能调优、Strage 模块的性能调优、Spark  Executor 的性能调优、Spark  SQL 的性能调优及 Spark 机器学习算法的性能调优等。

- Spark 生态圈

  包括常用重要技术框架如 Kafka、Docker 及 Alluxio （前
 Tachyon）等的原理与应用。

- Spark 应用开发案例

  在掌握 Spark 编程基础、内核原理、源码实现及生态圈的相关内容后，在实践中不断学习和积累 Spark 应用案例。

### 知识节点

- Scala 语言基础，包括 Scala 变量、基本类型、程序控制结构、集合、函数式编程、面向对象编程、模式匹配、类型系统、隐式转换、并发编程、Scala 与 Java 互操作等。

- Spark 简介，包括学习 Spark 的发展历史、“One Stack Rules Them All”设计思想，熟悉 Spark 生态圈、 Spark 核心组件等内容。

- Spark 开发环境部署：掌握 Hadoop 集群安装与部署、 Spark 集群安装与部署、Intelli IDEA  Spark 开发环境搭建、 Spark 应用程序本地调试与远程调试。

- Spark 编程模型：学会使用弹性分布式数据集（RDD）进行基础编程并掌握 RDD 运行原理，掌握 RDD 从创建、transformation、action、persist 及保存操作的整个生命周期过程，理解 RDD 的宽依赖与窄依赖、RDD 的 Lineage 关系及其对作业执行的影响等。

- Spark 工作机制： 掌握 Spark 整体架构、任务运行原理，掌握任务监控方法、广播变量与累加器的使用。

- Spark SQL：了解 Spark SQL 的发展历史及在 Spark 中的重要地位；掌握如何创建 DataFrame，包括从 RDD、外部数据源创建。熟练使用 DataFrame 的常用 API 包括 to、as、join、sort、orderBy 等方法，能够使用 SQL 语句替代 DataFrame 多种方法的组合完成相同功能；掌握如何创建 DataSet，熟练使用 DataSet 常用 API，包括 toDF、map、filter 等；理解 RDD、DataFrame、DataSet 间的异同；能够熟练使用常用数据源如 JSON、Parquet 等文件存储格式；能够进行多表关联使用复杂 SQL；理解 Spark SQL框架，对 LogicalPlan、Analyzed Logical Plan、Optimized Logical Plan、Physical Plan 等涉及到的相关类作用有清晰的认识；掌握 Spark SQL 执行过程包括全表查询、filter 查询、join 查询、子查询及聚合查询等的执行过程；掌握使用 Spark SQL 用户自定义函数。

- Spark Streaming：理解 Spark 流式计算的原理与使用方法；掌握 Spark Streaming 核心类如 StreamingContext 的使用、DStream 的原理与使用、DStream 常用输入数据源；掌握常用 DStream 操作如 Transformation、Windows 操作等，理解 DStream 的持久化与 Checkpoint 机制；能够掌握多种输入数据源的 Streaming 如文件输入数据源、Socket 输入数据源、Kafka 输入数据源等对应的流式应用；能够掌握 Spark Streaming 与 Kafka 集成使用方法；了解 DStream 和 DStreamGraph、DStream 和 RDD 间的关系；对 Spark Streaming 的容错机制如工作节点失效时的容错处理、Driver 节点失效时的容错处理有清晰的认识；掌握 DStream 作业的产生与调度原理，掌握 DStream 对应 Job 的生成、Job 的调度过程，清楚 Streaming 作业与 Spark 作业之间的关系。

- Spark Graphx：理解图计算的原理与应用场景；了解常用的图计算框架，了解 Spark 图计算框架、发展历程及自身特点；掌握 Spark  Graphx 属性图及常用图操作，掌握常用的图操作包括属性操作、结构操作、join 操作及 Neighborhood Aggregation 操作等；理解Vertex RDD 和 Edge RDDs 等 Gpraphx 中的重要数据结构；掌握图的边和顶点计数、图过滤、PageRank 算法、图的三角计数（Triangle Counting）、求图的连通组件（Connected Components）等常用图计算算法。

- Spark ML/MLLib：了解常用分布式机器学习算法框架及分布式环境下机器学习算法的特点；了解 Spark 机器学习组件，理解 ML 和 MLlib 的关系；掌握分类回归算法如 SVM、逻辑回归、线性回归、朴素贝叶斯、决策树、集成树（Ensembles of Trees）等的原理与应用；掌握协同过滤算法如 ALS 算法的原理与使用；掌握聚类算法如 K 均值、高斯混合模型、LDA 等算法的原理与使用；掌握奇异值分解（SVD）、主成分分析（PCA）降维算法的原理与使用；掌握特征提取与转换的使用；掌握常用的数学统计方法如假设检验、相关性统计等；理解机器学习算法的设计流程，掌握算法的特征提取、模型训练、模型使用及算法性能评价与调优等重要技能。

- Spark R：掌握 Spark R中的 Spark Context、SQLContext 的使用，掌握从本地 R Data Frame、数据源 JSON 等文件格式及现有 Hive 表上创建 DataFrame 的方法，学会使用 DataFrame 的基本操作如分组、聚合及查询行列数据等操作，掌握如何使用 Spark R 运行 
SQL 语句及如何在 Spark R 上使用机器学习算法如高斯 GLM 模型等。

- Spark 内核源码学习：掌握 Spark 任务调度模块源码包括 Spark Context、DAGScheduler、SchedulerBackend、TaskScheduler 等类的源码实现；掌握 Spark 运行模式（底层资源管理）包括 Local 模式、Spark Stanalone 模式、Mesos 模式、Yarn 模式等的实现，特别是 Spark Stanalone 模式下的源码实现；掌握 Spark Shuffle 模块包括 Hash Based Shuffle Write、Sort Based Write、Shuffe Read 等的源码实现；掌握 Spark Storage 模块包括 DiskStore、MemoryStore、Block 存储等相关类的源码实现。

- Spark MLlib/ML 源码分析：包括但不限于掌握分类回归算法如
 SVM、逻辑回归、线性回归、朴素贝叶斯、决策树、集成树（Ensembles of Trees）等的源码实现；掌握 ALS 算法的源码实现；掌握聚类算法如 K 均值、高斯混合模型、LDA 等算法的源码实现；掌握奇异值分解（SVD）、主成分分析（PCA）降维算法的源码实现。

- Spark Streaming 源码分析： 包括但不限于StreamingContext、DStreamGraph、DStream、JobScheduler、JobGenerator、ReceiverTracker、InputDStream、ReceivedBlockTracker、WAL、Receiver、ReceiverSupervisor、BlockGenerator 等核心类的源码实现。

- Spark Graphx 源码分析：包括但不限于 Graph、VertexRDDs、EdgeRDD、GraphLoader等核心类的源码实现，掌握PageRank、图的连通组件及三角计算等图算法的源码实现。

- Spark SQL 源码分析：包括但不限于 SQLContext、DataFrame、DataSets、QueryExecution、LogicalPlan 及 Spark Planner 等核心类的源码实现。 