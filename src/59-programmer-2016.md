## 基于 Spark 一栈式开发的通信运营商社交网络

文/丁廷鹤

本文从大数据思维角度重新审视运营商数据，利用业界前沿的 Spark 分析工具 Streaming、GraphX、MLlib 模块快速实现分布式图计算、流式计算以及机器学习技术。在大数据框架下构建基于运营商数据的实时社交网络，然后分析计算业务信息，同时探索性地开发了欺诈检测、社区发现、精准营销方面的服务，对于挖掘企业图结构数据中的价值具有一定的参考作用。

### 图计算在通信运营商行业的应用

社交网络是指人与人之间通过关系建立起来的网络结构，比如朋友、家谱、交易、通信等关系，为信息的交流与分享提供了便利。从社会学角度来看，它是指社会行动者及其之间的关系。通信运营商业务包含了大量用户电话通信、短信、位置等数据，这类数据实际暗含了用户之间的社交关系，构成一张巨大的社交网络结构，蕴含着极其宝贵的数据价值。这些社交关系都可以从属性图的角度看待，基于属性图对社交关系价值进行深层次挖掘，对一些传统的分析业务有巨大的帮助，甚至可能产生新的数据价值或赢利点。笔者主要是对用户通话数据进行的分析建模，构造实时的社交网络属性图，然后利用机器学习挖掘创新价值。
我们通过流计算、图计算、机器学习的方法实时建立了社交网络模型，初始做了欺诈预测的服务功能。后续我们在该社交网络还开发了许多功能，简单举例如图1。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581809541f0b3.png" alt="图1  基于社交网络提供的部分功能服务" title="图1  基于社交网络提供的部分功能服务" />

**图1 基于社交网络提供的部分功能服务**

从图1可以看出，首先我们做了一些基础的统计分析服务，这些服务对用户画像或数据统计提供了高效快速的支持，还利用算法进行了机器学习相关研究，如利用 StreamingKmeans 的骚扰电话、利用 PageRank 的 VIP 用户排名、利用 LabelPrepagation 的社区发现、利用 SVDPlusPlus、PersonalRank、二跳结点的推荐服务，这些应用研究对于欺诈预测、用户分析、精准营销提供了参考。

从社交网络到语言模型、知识图谱，日益重要且价值巨大的图数据驱动着多图并行存储与计算系统，比如 Pregel、GraphLab 和 GraphX，对图数据挖掘及机器学习都有着重要的推动作用。本项目就是利用 Spark GraphX、MLlib、Streaming 对通信运营商数据做一栈式开发处理。Spark 最初是2009年伯克利大学 AMP 实验室的一个研究项目，2010年开源供全世界的工程师学习使用。Spark 在 GitHub 上开源以后发展了自己的开发者社区，直到2013年转至 Apache 下成为开源项目。截止今日，已经有超过200个公司、1000多位工程师为 Spark 发展贡献了力量。GraphX 是基于 Spark 引擎开发的分布式图存储和图并行计算模块，它在比较高的层面扩展了 RDD 概念，提出图的抽象含义——具有属性的点和边构成的有向图。为了支持图计算，GraphX 提出了一系列基础的核心操作来实现 Pregel 功能以及复杂的操作组合，同时也实现了查询优化。另外，GraphX 提供的图算法还便于图分析任务的应用。GraphX 在丰富性、易用性、性能之间作了较好的平衡。

### 分布式图计算

#### 属性图

GraphX 提供了统一的抽象概念，在不做数据的移动或者数据重复操作下可将相同的数据看成图结构或者表结构。GraphX 不仅提供了一系列标准的图并行操作，包括 MapReduce、Filter、Join 等，还提供了一系列复杂操作如 Subgraph 等。GraphX 以 Scala 为核心语言，以 Spark 为底层核心，具有编程方便的极大优点，通过对边与点的操作，即可快速地创建图系统。我们先介绍下属性图的概念，属性图是一种有向的多重图，两点之间可以允许有多个边共享相同结点。因此，属性图是由点及点之间的边组成的，点是由用户指定的主键以及属性组成，边由两个点的主键及属性组成。跟 RDD 一样，属性图不可改变，是分布式、有容错功能的。改变一个属性图的值或结构会直接生成一个新的属性图，不过系统会自动重用之前的属性图加快新属性图的生成。当属性图出现故障时可以在不同的集群节点上快速恢复。下面利用代码定义了一个基本的属性图：

```
class Graph[VD, ED] {
val vertices: VertexRDD[VD]
val edges: EdgeRDD[ED]
}
```

**代码1**

VertexRDD[VD]以及 EdgeRDD[ED]分别扩展了 RDD 结构中的RDD[(VertexID, VD)] 及 RDD[Edge[ED]]，但是提供了一些附加的操作及优化功能。VertexRDD 有一些核心操作，相应函数如下：

```
class VertexRDD[VD] extends RDD[(VertexID, VD)] {
// Filter the vertex set but preserves the internal index
def filter(pred: Tuple2[VertexId, VD] => Boolean): VertexRDD[VD]
// Transform the values without changing the ids (preserves the internal index)
def mapValues[VD2](map: VD => VD2): VertexRDD[VD2]
def mapValues[VD2](map: (VertexId, VD) => VD2): VertexRDD[VD2]
// Show only vertices unique to this set based on their VertexId's
  def minus(other: RDD[(VertexId, VD)])
  // Remove vertices from this set that appear in the other set
  def diff(other: VertexRDD[VD]): VertexRDD[VD]
  // Join operators that take advantage of the internal indexing to accelerate joins 
  def leftJoin[VD2, VD3](other: RDD[(VertexId, VD2)])(f: (VertexId, VD, Option[VD2]) => VD3): VertexRDD[VD3]
  def innerJoin[U, VD2](other: RDD[(VertexId, U)])(f: (VertexId, VD, U) => VD2): VertexRDD[VD2]
  // Use the index on this RDD to accelerate a `reduceByKey` operation on the input RDD.
  def aggregateUsingIndex[VD2](other: RDD[(VertexId, VD2)], reduceFunc: (VD2, VD2) => VD2): VertexRDD[VD2]
}
```

**代码2**

相应的 EdgeRDD 同样有一系列核心操作，如下：

```
// Transform the edge attributes while preserving the structure
def mapValues[ED2](f: Edge[ED] => ED2): EdgeRDD[ED2]
// Reverse the edges reusing both attributes and structure
def reverse: EdgeRDD[ED]
// Join two `EdgeRDD`s partitioned using the same partitioning strategy.
def innerJoin[ED2, ED3](other: EdgeRDD[ED2])(f: (VertexId, VertexId, ED, ED2) => ED3): EdgeRDD[ED3]
```

**代码3**

我们通过一个例子来说明属性图结构，某大学有很多人员，可以分为学生、教授、博士后等身份，可以看做是属性图中点结构；人员之间又有着同事、访问、合作等关系，可以看做是属性图中边结构。图2用表格及有向图表示了这个例子。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58180c30817b8.png" alt="图2  根据大学人员关系建立的属性图" title="图2  根据大学人员关系建立的属性图" />

**图2 根据大学人员关系建立的属性图**

构造此属性图的 Scala 代码如下：

```
// Assume the SparkContext has already been constructed
val sc: SparkContext
// Create an RDD for the vertices
val users: RDD[(VertexId, (String, String))] = sc.parallelize(Array((3L, ("rxin", "student")), (7L, ("jgonzal", "postdoc")), (5L, ("franklin", "prof")),  (2L, ("istoica", "prof"))))
// Create an RDD for edges
val relationships: RDD[Edge[String]] =sc.parallelize(Array(Edge(3L, 7L, "collab"),    Edge(5L, 3L, "advisor"), Edge(2L, 5L, "colleague"), Edge(5L, 7L, "pi")))
// Define a default user in case there are relationship with missing user
val defaultUser = ("John Doe", "Missing")
// Build the initial Graph
val graph = Graph(users, relationships, defaultUser)
```

**代码4**

如例子所示，属性图中的点结构为点 ID 以及点属性组成，边结构为源点ID、目的点ID以及边属性组成，边的两个 ID 指示出了边的方向。我们可以对边或者点分别做操作，比如：

```
val graph: Graph[(String, String), String] // Constructed from above
// Count all users which are postdocs
graph.vertices.filter { case (id, (name, pos)) => pos == "postdoc" }.count
// Count all the edges where src > dst
graph.edges.filter(e => e.srcId > e.dstId).count
```

**代码5**

属性图中还有一个非常重要的结构——Triplet，它由两个点以及点之间的边组成，用 SQL 语言可以表示如下：

```
SELECT src.id, dst.id, src.attr, e.attr, dst.attr
FROM edges AS e LEFT JOIN vertices AS src, vertices AS dst
ON e.srcId = src.Id AND e.dstId = dst.Id
```

**代码6**

用图形表示 Triplet 见图3。

我们可以利用 Triplet 结构来进行关系点间的数据分析，比如：

```
val graph: Graph[(String, String), String] // Constructed from above
// Use the triplets view to create an RDD of facts.
val facts: RDD[String] = graph.triplets.map(triplet =>
  triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1)
facts.collect.foreach(println(_))
```

**代码7**

#### 图操作
GraphX 设计了两个类 Graph 和 GraphOps 的图操作。一些底层优化后的核心操作在 Graph 里面进行了定义。利用核心操作设计的高级操作则放在 GraphOps 里供使用。我们可以将图的操作大体分为以下几类：

- Property Operators

- Structural Operators

- Join Operators

- Aggregate Messages

- Computing Degree Information

- Collecting Neighbors

- Caching and Uncaching

下面对每种操作都做简单介绍，首先是 Property Operators 对图的组成结构所做的转换，如下：

```
class Graph[VD, ED] {
  def mapVertices[VD2](map: (VertexId, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
}
```

**代码8**

Structural Operators 操作对图的整体结构做了一些转换，比如图反向操作 Reverse、计算子图的操作 Subgraph、计算两图相交子图的操作 Mask 以及对边做聚合的操作 GroupEdges，相应的 API 定义如下：

```
class Graph[VD, ED] {
  def reverse: Graph[VD, ED]
  def subgraph(epred: EdgeTriplet[VD,ED] => Boolean,
    vpred: (VertexId, VD) => Boolean): Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD,ED]
}
```

**代码9**

Join Operators 操作是属性图跟其他结构做的 Join 操作，比如：

```
class Graph[VD, ED] {
  def joinVertices[U](table: RDD[(VertexId, U)])(map: (VertexId, VD, U) => VD)
    : Graph[VD, ED]
  def outerJoinVertices[U, VD2](table: RDD[(VertexId, U)])(map: (VertexId, VD, Option[U]) => VD2): Graph[VD2, ED]
}
```

**代码10**

Aggregate Messages 操作是图计算里一个非常重要的操作，许多算法 PageRank、最短路径、连通图、标签传播等都在其基础上开发。它定义了相连结点之间数据传输、接收等过程，包含两个重要的函数——发送消息函数 SendMsg 将定义的数据从一个结点发送到相连的另一个结点，接收消息函数 MergeMsg 将结点收到的所有消息按一定规则进行处理分析。当传递的 Message 占用空间是常数大小时，该操作的性能表现最佳。AggregateMessages API 定义如下：

```
class Graph[VD, ED] {
  def aggregateMessages[Msg: ClassTag](
      sendMsg: EdgeContext[VD, ED, Msg] => Unit,
      mergeMsg: (Msg, Msg) => Msg,
      tripletFields: TripletFields = TripletFields.All
): VertexRDD[Msg]
}
```

**代码11**

下面通过计算比某用户年龄大的所有关注者的平均年龄来展示AggregateMessages函数作用：

```
import org.apache.spark.graphx.{Graph, VertexRDD}
import org.apache.spark.graphx.util.GraphGenerators
// Create a graph with "age" as the vertex property.
// Here we use a random graph for simplicity.
val graph: Graph[Double, Int] =
  GraphGenerators.logNormalGraph(sc, numVertices = 100).mapVertices( (id, _) => id.toDouble )
// Compute the number of older followers and their total age
val olderFollowers: VertexRDD[(Int, Double)] = graph.aggregateMessages[(Int, Double)](
  triplet => { // Map Function
    if (triplet.srcAttr > triplet.dstAttr) {
      // Send message to destination vertex containing counter and age
      triplet.sendToDst(1, triplet.srcAttr)
    }
  },
  // Add counter and age
  (a, b) => (a._1 + b._1, a._2 + b._2) // Reduce Function
)
// Divide total age by number of older followers to get average age of older followers
val avgAgeOfOlderFollowers: VertexRDD[Double] =
olderFollowers.mapValues( (id, value) =>
    value match { case (count, totalAge) => totalAge / count } )
// Display the results
avgAgeOfOlderFollowers.collect.foreach(println(_))
```

**代码12**

Computing Degree Information 操作给出了计算图里每个结点出度、入度、度的计算，下面展示了利用 Degree API 计算最大度的功能：

```
// Define a reduce operation to compute the highest degree vertex
def max(a: (VertexId, Int), b: (VertexId, Int)): (VertexId, Int) = {
  if (a._2 > b._2) a else b
}
// Compute the max degrees
val maxInDegree: (VertexId, Int)  = graph.inDegrees.reduce(max)
val maxOutDegree: (VertexId, Int) = graph.outDegrees.reduce(max)
val maxDegrees: (VertexId, Int)   = graph.degrees.reduce(max)
```

**代码13**

Collecting Neighbors 操作用来统计结点的邻居，这是一个消耗非常大的操作，如果可以的话最好使用 AggregateMessages API。其 API 定义如下：

```
class GraphOps[VD, ED] {
  def collectNeighborIds(edgeDirection: EdgeDirection): VertexRDD[Array[VertexId]]
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[ Array[(VertexId, VD)] ]
}
```

***代码14**

Caching 和 Uncaching 操作主要是在图计算过程中的优化操作，如果某个图后续需要计算多次，那么开始时可以利用 Cache 将其放入内存加快运算；如果某个图 Cache 后计算完，可使用 Uncache 操作将其从内存中释放为其他数据存储。

#### 图优化

GraphX 采用的是图存储中点分割的方案。这种方案的设计理念是边唯一存储、点冗余存储，每个边只会在一台机器上存储，而同一个结点则可能会出现在不同机器上面。如果对节点进行更新，则首先更新一个主结点数据，然后对其他节点进行更新。在做点与点间的操作时，可以同时在不同机器上进行，最后将结果进行汇总，网络开销比较小。代价是每个点要存储多次，点的更新也有数据同步的开销。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581813eb28cbf.png" alt="图4  边分割与点分割的切割方式" title="图4  边分割与点分割的切割方式" />

**图4  边分割与点分割的切割方式**

#### 图算法

GraphX 提供了一系列算法操作，比如 PageRank，Connected Components 和 Triangle Counting。

PageRank 是 Google 创始人拉里·佩奇和谢尔盖·布林于1997年构建搜索系统原型时提出的链接分析算法，自 Google 在商业上获得空前的成功后，该算法也成为其他搜索引擎和学术界十分关注的计算模型。PageRank 算法有两个过程，首先给每个结点定义一个初始权重，其次每个结点将其权重平均分配到其出边，同时将入边上的权重按一定规则汇总至本身，然后迭代稳定到相关的阈值。下面是一个算法实例：

```
import org.apache.spark.graphx.GraphLoader
// Load the edges as a graph
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")
val ranks = graph.pageRank(0.0001).vertices
// Join the ranks with the usernames
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val ranksByUsername = users.join(ranks).map {
  case (id, (username, rank)) => (username, rank)
}
// Print the result
println(ranksByUsername.collect().mkString("\n"))
```

**代码15**

接下来我们构造一个属性图，通过计算子图以及 PageRank，求出权重较大结点的邻居。

```
import org.apache.spark.graphx.GraphLoader
// Load my user data and parse into tuples of user id and attribute list
val users = (sc.textFile("data/graphx/users.txt")
  .map(line => line.split(",")).map( parts => (parts.head.toLong, parts.tail) ))
// Parse the edge data which is already in userId -> userId format
val followerGraph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")
// Attach the user attributes
val graph = followerGraph.outerJoinVertices(users) {
  case (uid, deg, Some(attrList)) => attrList
  // Some users may not have attributes so we set them as empty
  case (uid, deg, None) => Array.empty[String]
}
// Restrict the graph to users with usernames and names
val subgraph = graph.subgraph(vpred = (vid, attr) => attr.size == 2)
// Compute the PageRank
val pagerankGraph = subgraph.pageRank(0.001)
// Get the attributes of the top pagerank users
val userInfoWithPageRank = subgraph.outerJoinVertices(pagerankGraph.vertices) {
  case (uid, attrList, Some(pr)) => (pr, attrList.toList)
  case (uid, attrList, None) => (0.0, attrList.toList)
}
println(userInfoWithPageRank.vertices.top(5)(Ordering.by(_._2._1)).mkString("\n"))
```

**代码16<**

Connected Components 可以找出图中存在的连通图。Triangle Counting 可以计算出图中的三角数目。

### 通信运营商社交网络开发简介

首先简单介绍我们的项目架构（以最复杂的欺诈预测为例）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58181460b772b.png" alt="图5  项目一期的架构图" title="图5  项目一期的架构图" />

**图5 项目一期的架构图**

我们项目一期的底层架构分为三层，分别是 ETL 层、分布式图计算层、机器学习层。ETL 层主要是对原始的离线数据做清洗、过滤、转换工作，以及对增量数据做实时清洗、过滤和转换。我们对增量数据做的实时 ETL 工作建立在 Spark Streaming 框架之上，复用了对离线数据处理的一些函数。本项目可以看做是利用 Spark Streaming、GraphX、MLlib 模块做的一栈式开发，融入了流处理、分布式图计算以及机器学习的技术。对增量数据处理的 Streaming 部分，首先消费 Kafka 消息生成 RDD 数据，然后调用函数实现：

```
this.dataStream.foreachRDD { rdd =>
if (!rdd.isEmpty) {
       
}
}
```

**代码17**

对于分布式图计算层，我们分别利用处理好的数据建立属性图，在实时处理部分，我们会做属性图的 Join 操作来实现图的实时更新。建立属性图的部分代码为：

```
def createGraph(newRdd:RDD[String]): GraphEntity ={
  val vertex: VertexEntity = this.vertexFactory.createVertex(newRdd)
  val edge: EdgeEntity = this.edgeFactory.createEdge(newRdd)
  this.createGraph(vertex, edge)
}
```

**代码18**

利用实时计算，建立实时更新的图模型核心代码为：

```
def unionGraphEntity(otherGraphEntity: GraphEntity): Graph[Array[String], Array[String]] = {
val newVertex = this.graphRDD.vertices.union(otherGraphEntity.graphRDD.vertices)
  val newEdge = this.graphRDD.edges.union(otherGraphEntity.graphRDD.edges)
  Graph(newVertex, newEdge)
}
```

**代码19**

本项目代码量较多，底层实现了很多方法，在此不一一叙述。当建立属性图后，我们在图中抽取特征并做特征的标准化等操作（项目一期没有标签数据，因此我们解决无监督预测的问题，在此步骤通过对出入度设定阈值筛选了部分高可疑用户后去做下一步的聚类）。然后将特征包装成了实时的流式特征，核心代码为：

```
def packageStreamData(ssc: StreamingContext, graphFactory:GraphFactory, newFilesPath: String){
  this.testDataQueue = mutable.Queue( this.scaledTrainData )
  this.trainDataQueue = mutable.Queue( this.scaledTrainData.map(_._2)     this.testDataStream = ssc.queueStream(testDataQueue, true, null)
  this.trainDataStream = ssc.queueStream(trainDataQueue, true, null)
  this.dataStream = graphFactory.getDataStream(ssc, newFilesPath)
}
def updatePackageData(ssc: StreamingContext, graphFactory: GraphFactory, graph: GraphEntity, threshold: Int) = {
  this.dataStream.foreachRDD { rdd =>
    if (!rdd.isEmpty) {
      val streamCommunicationNetwork = graphFactory.createGraph(rdd)
    graph.setGraphRDD(graph.unionGraphEntity(streamCommunicationNetwork))
      val bulkingData: RDD[(VertexId, Vector)] = graphFactory.extractStreamingFeature(graph, streamCommunicationNetwork, threshold)
      this.trainData = this.trainData.union(bulkingData)
      val scaledBulkingData: RDD[(VertexId, Vector)] = this.standScalerData(bulkingData)
      this.trainDataQueue += scaledBulkingData.map(_._2)
      this.testDataQueue += scaledBulkingData
      println("Next bash data analyse:")
    }
  }
}
```

**代码20**

有了这种流式特征后，用 StreamingKmeans 训练，然后做欺诈预测，训练的核心代码为：

```
private var model: StreamingKMeans = null
def initModel = {
  this.model = new StreamingKMeans()
    .setK(clusterNumber)
    .setDecayFactor(threshold)
    .setRandomCenters(dim, weight)
}
def train(trainDataStream: DStream[Vector]) = this.model.trainOn(trainDataStream)
```

**代码21**

该算法可以分为离线学习及在线学习两类，离线学习是经典的 Kmeans 算法，过程如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581812bb6989f.png" alt="图6  Kmeans的算法原理" title="图6  Kmeans的算法原理" />

**图6 Kmeans 的算法原理**

算法的在线学习部分，主要是对聚类的中心进行处理，公式如下：

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58181275aff2b.png" alt="" title="" />

公式中 n 为离线数据量，m 为增量数据量，c 与 x 分别为离线数据与增量数据的聚类中心，a 则为算法的学习速率，a 越小，增量数据影响越大，反之成立。

通过上面列举的部分核心代码，建立实时更新的分布式图模型，并通过筛选用户提取特征后应用了流式聚类的算法。我们后期数据预测结果抽样了一小部分数据，通过验证得出 Precision 在0.94以上（此处从业务角度考虑并不需要过多追求 Recall 的值）。当然我们一期基于无监督算法做出来的模型效果没有较强的业务应用说服力，但是对于推动通信运营商通话业务中存在的欺诈行为起了标新立异的效果。项目二期我们也是通过多渠道采集标签数据，将模型流程改成了离线训练在线预测的模式，算法也是转向了有监督学习，在此不再过多介绍。

接下来，我们针对其中的共同好友、SVDPlusPlus 及 LabelPrepagation 的实现做核心介绍。共同好友利用了属性图中非常重要的两个功能：消息聚合和 Triplet 结构。首先利用 AggregateMessages 函数将每个结点的 ID 发送给邻居，同样每个结点接收邻居发送的 ID 保存到 List 变量中，接着通过 Triplet 结构做两个结点之间邻居 List 变量的交集。其核心代码如下：

```
private def getAdjacencyVertex(): RDD[(VertexId, List[Double])] = this.graphRDD.aggregateMessages[List[Double]](
    triplet => {
      triplet.sendToDst(List(triplet.srcId))
      triplet.sendToSrc(List(triplet.dstId))
    },
    (oneMethod, anotherMethod ) => oneMethod ++ anotherMethod
  ).map(data => (data._1, data._2.distinct.sorted))
  def computeMutualVertex(): RDD[(VertexId, VertexId, List[Double])] ={
    val vertex = this.getAdjacencyVertex()
      val distinctEdges: RDD[Edge[Double]] = this.uniqueGraphRDD.edges
    val mutualGraph = Graph(vertex, distinctEdges)
    val mutualVertex = mutualGraph.triplets.map{triplet =>
      val mutualVertexSet = triplet.srcAttr intersect(triplet.dstAttr)
      (triplet.srcId, triplet.dstId, mutualVertexSet)
    }
    mutualVertex
}
```

**代码22**

SVDPlusPlus 算法是基于协同过滤的推荐算法，SVD 算法是在 Baseline Estimates 的基础上加入了 Latent Factor 取得了较好的效果，SVDPlusPlus又在SVD的基础上加入了 Implicit Feedback 大大提高了预测的准确性。

其数学公式为：

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58181341e8de8.png" alt="" title="" />

LabelPrepagation 算法则贯彻了“人以类聚，物以群分”理念，将不同的用户划分到不同社区，同一社区的用户有着较为相近的特性，其核心实现代码为：

```
def run[VD, ED: ClassTag](graph: Graph[VD, ED], maxSteps: Int): Graph[VertexId, ED] = {
require(maxSteps > 0, s"Maximum of steps must be greater than 0, but got ${maxSteps}")
val lpaGraph = graph.mapVertices { case (vid, _) => vid }
def sendMessage(e: EdgeTriplet[VertexId, ED]): Iterator[(VertexId, Map[VertexId, Long])] = {
Iterator((e.srcId, Map(e.dstAttr -> 1L)), (e.dstId, Map(e.srcAttr -> 1L))
}
def mergeMessage(count1: Map[VertexId, Long], count2: Map[VertexId, Long])
: Map[VertexId, Long] = {
(count1.keySet ++ count2.keySet).map { i =>
val count1Val = count1.getOrElse(i, 0L)
val count2Val = count2.getOrElse(i, 0L)
i -> (count1Val + count2Val)
}.toMap
}
}
```

**代码23**

本文简要介绍了社交网络的概念，给出了 Spark GraphX 一些基本的功能代码，并讲解了通信运营商利用该技术做的一栈式开发。我们除了构造了基础的实时图模型，还在此基础上做出了一些基础功能以及创新应用。我们坚信随着团队技术的积累，后续肯定会有更好的项目实施，本文在繁重的工作下写就，希望微小的工作能对各位读者有一些小的启发或借鉴。