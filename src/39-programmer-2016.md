## Impala 的信息仓库：解读 TQueryExecRequest 结构

文/查锐

相信很多做海量数据处理和大数据技术研发的朋友对 Impala 这个基于 Hadoop 的交互式 MPP 引擎都不陌生，尤其对 Impala 出色的数据处理能力印象深刻。在查询执行的整个生命周期内，Impala 主要通过 Frontend 生成优化的查询计划，Backend 执行运行时代码生成来优化查询效率。在客户端的一个 SQL 查询下发到 ImpalaServer 后，Frontend 会在生成查询计划的过程中，收集必要的统计信息，作为 Backend 分布式执行的依据。这些信息主要包括：表结构、分区统计、SQL 语句的表达式集合，以及执行计划分片描述等。这些信息的集合在 Impala 中被称为 TQueryExecRequest。通过名字可以看出，这是个 Thrift 结构，由 Frontend 封装，通过 Impala 服务开启的 ThriftServer 发送到后端的 Coordinator。可以说，TQueryExecRequest 结构就是 Impala 执行查询的信息仓库，熟悉这个结构有助于理解整个 Impala 的分布式架构实现。

### TQueryExecRequest 结构组成

TQyeryExecRequest 结构中，主要包含如下成员：

```
TDescriptorTable desc_tbl
vector<tplanfragment> fragments
vector<int32_t> dest_fragment_idx
map<tplannodeid, vector<tscanrangelocations=""> > per_node_scan_ranges
TResultSetMetadata result_set_metadata
TFinalizeParams finalize_params
TQueryCtx query_ctx
string query_plan
TStmtType::type stmt_type
int64_t per_host_mem_req
int64_t per_host_vcore
vector<tnetworkaddress> host_list
string lineage_graph</tnetworkaddress></tplannodeid,></int32_t></tplanfragment>
```

#### 描述符表

描述符表的类型为 TDescriptorTable，包含和查询结果相关的 table、tuple 以及 slot 描述信息。

这里需要说明的是，无论提交的 SQL 查询有多复杂，包括数据过滤、聚合、JOIN，都是在对 HDFS 或 HBase（所查询分区内的）全量数据扫描的基础上进行的。数据扫描结果集的每一行都是一个 tuple，tuple 中的每个字段都是一个 slot。描述符表有如下成员：

```
vector<tslotdescriptor> slotDescriptors
vector<ttupledescriptor> tupleDescriptors
vector<ttabledescriptor> tableDescriptors</ttabledescriptor></ttupledescriptor></tslotdescriptor>
```

#### Tuple 描述符

在 Impala 中，一个查询返回结果集中的每一行叫做一个 tuple，对 tuple 的描述信息存放在 tuple 描述符结构中。tuple 描述符有如下成员：

```
TTupleId id;
int32_t byteSize;
int32_t numNullBytes;
TTableId tableId;
```

1. id 是表中的 tuple id，截止到 Impala2.2 版本，一个 tupleRow 中只会有一个 tuple，因此 id 总是0。
2. byteSize 是一个 tuple 中所有 slot 大小的和，再加上 nullIndicator 占用的字节数。
3. numNullBytes 指的是，如果 tuple 中的一个 slot 可以为 NULL，则需要占用 tuple 1个 bit 大小的空间，numNullBytes 等于(N + 7) / 8，N 是可能为 NULL 的 slot 个数。例如在一个 tuple 中，可能为 NULL 的 slot 数为0，则 numNullBytes=0，也就是不需要额外的空间去存储 null indicator 的信息；如果可以为 NULL 的 slot 数为1-8，则 numNullBytes 为1。
4. tableId 是 tuple 所属的 table id。

#### Slot 描述符

slot 描述信息存放在 TSlotDescriptor 结构中，主要用来存放所查询表中某个字段的相关信息，成员包括：

```
TSlotId id
TTupleId parent
TColumnType slotType
vector<int32_t> columnPath
int32_t byteOffset
int32_t nullIndicatorByte
int32_t nullIndicatorBit
int32_t slotIdx
bool isMaterialized</int32_t>
```

id 在一个 tuple 中唯一标识一个 slot。

parent 代表 slot 所在的 tuple id。

slotType 是 slot 所在的 column 类型。对于 Parquet 嵌套文件格式来说，一个 TColumnType 可能有多个 TTypeNode。例如，一个 STRUCT 类型可能包含多个 SCALAR 类型。一个 TTypeNode 的类型可能是标量的（确定的内建类型）、数组（ARRAY）、映射（MAP）或者结构（STRUCT）。目前 Impala 的2.3版本已经支持 ARRAY、MAP 和 STRUCT 这三种复杂类型（仅限 Parquet 文件格式），这个新特性应该会使更多的人转向 Impala 阵营。TColumnType的结构如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab15e5e2f2f.jpg" alt="图1  TColumnType结构图" title="图1  TColumnType结构图" />

图1  TColumnType 结构图

columnPath 是一个整型集合，但是在前端只会填充一个元素，因为一个 slot 只对应一个 columnPath。columnPath 代表一个 slot 在表中的位置，如 create table t (id int, name string)，则 id 的 columnPath 为0，name 的 columnPath 为1。对于分区表，partition key 字段的 columnPath 会排在前面。如 create table t（id int、name string、calling string）partitioned by (date string, phone int)，则 date 的columnPath 为0， phone 的 columnPath 为1， id 的 columnPath 为2，name 的 columnPath 为3，calling 的 columnPath 为4。在查询时，columnPath 的顺序是按照 column 在提交的 SQL 语句中出现的顺序排列的，如 select name from t where date = '2016-01-01' and calling = '123' and phone = abs(fnv_hash('123')) % 10这个 SQL，columnPath 的顺序为3, 0, 4, 1。

byteOffset 是 slot 在 tuple 中的偏移，单位为字节。

nullIndicatorByte 表明当前 slot 为 NULL 时，在 tuple 的哪个字节中。

nullIndicatorBit 表明当前 slot 为 NULL 时，在第 nullIndicatorByte 个字节的哪个 bit 上。

slotIdx 是 slot 在 tuple 中的序号。

isMaterialized 表明当前 slot 是否被物化。对于 partition key，也就是 clustering column 来说，isMaterialized 为 false，也就是 partition key 不会被物化。

#### Table 描述符

Table 描述符包含和查询相关表的信息，包括表字段、类型、分区等信息。成员包括：

```
TTableId id;
TTableType::type tableType;
int32_t numCols;
int32_t numClusteringCols;
vector<string> colNames;
THdfsTable hdfsTable;
THbaseTable hbaseTable;
TDataSourceTable dataSourceTable;
string tableName;
string dbName;</string>
```

在一个查询中，id 字段唯一标识一个表。

tableType 表明表的类型，TTableType 为枚举类型，包括 HDFS_TABLE = 0，HBASE_TABLE = 1，VIEW = 2，DATA_SOURCE_TABLE = 3。

numCols 代表 table 中 column 的个数。

numClusteringCols 代表 table 中 clustering column 的个数，也就是 parititon 的个数。

colNames 代表 table 中所有 column 的名称的集合。

#### THdfsTable 中的分区信息

THdfsTable 结构中包含当前 table 和 HDFS 相关的所有信息。关于 hdfs table 中的 partition 信息，有如下说明：

partition key 的类型只能是标量的，如 int、float、string、decimal、timestamp。
不同的 partition 可以有不同的的文件格式，用户可以在一个表中，增加、删除分区，为分区设置特定的文件格式：

```
[localhost:21000] > create table census (name string) partitioned by (year smallint);
[localhost:21000] > alter table census add partition (year=2012); -- text format
[localhost:21000] > alter table census add partition (year=2013); -- parquet format
[localhost:21000] > alter table census partition (year=2013) set fileformat parquet;
```

THdfsTable 结构中的 partitions 类型为 map<int64_t, THdfsPartition>，这个字段不能为空，即使没有为表指定分区，也会有一个默认的 partition。

partitions 的 Size 为表中 partition 的总数。例如，一个表按 month、day 以及 postcode 分区，有12个 month，每个 month 中30个 day，每个 day 中100个 postcode，则 partition 数为12 * 30 * 100=36000（如上面的例子一个表按 month、day 以及 postcode 分区，month 为 string 类型，day 为 int 类型，postcode 为 bigint 类型，则 partitionKeyExprs 中的三个成员（TExpr）中的 nodes 的唯一一个元素（TExprNode）的 node_type 分别为 STRING_LITERAL(11)、INT_LITERAL(4) 和 INT_LITERAL(4)）。
THdfsPartition 结构中的 partitionKeyExprs 类型为 TExpr 的集合，每个 parititon key 的信息由一个 TExpr 描述。这里的 TExpr 的类型是标量类型（partition key 只能是标量类型）的字面值，基类为 LiteralExpr，根据不同的 partition key 类型，可能是 StringLiteral、IntLiteral 等。

TExpr 的成员类型是 TExprNode 的集合，由于 partition key 的类型为 LiteralExpr，所以这里的 TExprNode 集合中只会有一个成员，因为在 Expr 树中，LiteralExpr 节点不会再有孩子 Expr。

THdfsPartition 结构中的 file_desc 成员类型为 THdfsFileDesc 的集合，这表明一个分区下可能会有多个文件。THdfsFileDesc 结构中的 file_blocks 成员类型为 THdfsFileBlock 的集合，这表明一个文件可能由多个 block 组成，在 THdfsFileBlock 中指定了 block 的大小、在文件中的偏移量等。

THdfsTable 结构如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab163c94e5d.jpg" alt="图2  THdfsTable结构图" title="图2  THdfsTable结构图" />

图2  THdfsTable 结构图

#### 执行计划分片

一个查询请求提交给 ImpalaServer 之后，会在后端调用 JNI 初始化一个前端的 Frontend 实例，由这个实例对提交的 SQL 语句做语法分析，找出和查询相关的 table、scanList 以及 expr 信息，通过这些信息构造执行节点（如 HdfsScanNode、AggreagtionNode、HashJoinNode），根据节点类型评估节点为分布式执行还是本地执行，执行节点组成执行计划分片，最终构造出整个执行计划。

例如，客户端提交了一个 SQL 请求 select date, count(user) from t group by date，通过前端语法分析，可以得到如下信息：

1. scanList 由 date 和 user 字段组成。
2. groupingExpr 是一个 SLOT_REF 类型的表达式（date 字段）。
3. aggregateExpr 是一个 AGGREGATE_EXPR 类型的表达式（count(STRING)），它的孩子是一个 SLOT_REF 类型的表达式（user 字段）。
4. 
根据这些信息可以得到如图3所示的执行计划。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab16681d192.jpg" alt="图3  Frontend生成的执行计划" title="图3  Frontend生成的执行计划" />

图3  Frontend 生成的执行计划

下面分析 TQueryExecRequest 中的执行计划分片（fragments）结构。fragments 是一个TPlanFragment 类型的集合，一个 TPlanFragment 中包含和一个执行计划分片相关的所有信息。TPlanFragment 结构的成员如下：

```
string display_name
TPlan plan
vector<TExpr> output_exprs
TDataSink output_sink
TDataPartition partition
```

#### 执行计划分片中的执行节点信息

TPlanFragment 结构中的 plan 成员的类型为 TPlan，包含一个执行分片中的节点及其相关的表达式信息。TPlan 结构如图4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab1693940e5.jpg" alt="图4  TPlan结构中的执行节点信息" title="图4  TPlan结构中的执行节点信息" />

图4  TPlan 结构中的执行节点信息

需要说明的是，TPlan 的成员是 TPlanNode 的集合，每个 TPlanNode 包含一个执行节点的信息。

#### 全局 conjuncts

TPlanNode 结构中的 conjuncts 成员包含 where 子句的过滤条件，是 TExpr 类型的集合，而 TExpr 的成员又是 TExprNode 类型的集合。也就是说，conjuncts 包含了至少一棵表达式树，表达式树的信息由 TExpr 描述，树中的每个节点的信息由 TExprNode 描述。表达式树的叶子节点的表达式类型一般是 SLOT_REF 或者 LITERAL，非叶子节点的表达式类型一般是 FUNCTION_CALL 或者 PREDICATE。FUNCTION_CALL 可能是内建的，也可能是 Hive 或者 Impala 的 UDF；PREDICATE 可能是 COMPOUND_PRED(and 和 or)、LIKE_PRED(a like '%b%')或者 IN_PRED(a in (1, 2, 3))等。例如，where 子句中的过滤条件为 phone = '123' OR imsi IS NOT NULL，则这个 conjuncts 树如图5所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab16c837734.jpg" alt="图5  组成Conjuncts的表达式树" title="图5  组成Conjuncts的表达式树" />

图5  组成 Conjuncts 的表达式树

#### 聚合操作节点 TAggregationNode

如果 TPlanNode 是一个 TAggregationNode，那么在 TAggregationNode 这个结构中有两个比较重要的字段，一个是 grouping_exprs，另一个是 aggregate_functions。grouping_exprs 的类型是一个 TExpr 集合，存储了至少一棵 TExpr 树。grouping_exprs 中的每棵 TExpr 树描述了一个分组（group），例如 group by fnv_hash(date)分组中的 fnv_hash(phone)就是在 grouping_exprs 的一棵 TExpr 树中描述的，树的叶子节点的表达式类型为 SLOT_REF，父节点为 FUNCTION_CALL。和 grouping_exprs 类似的是，aggregate_functions 也是一个 TExpr 集合，只不过它描述的是聚合函数的信息，例如聚合函数 count(user)在 aggregate_functions 的一棵 TExpr 数中描述，叶子节点的表达式类型为 SLOT_REF，父节点表达式类型为 AGGREGATE_EXPR。

对于一个聚合操作来说，执行计划的最底层两个分片都会包含 AggregationNode，但是这两个AggregationNode 的 grouping_exprs 和 aggregate_functions 中的 TExprNode 节点类型以及节点的字面值（scalar_type）类型不尽相同。第0个分片中的聚合操作是 UNPARTITIONED 的，也就是说当前聚合操作的结果要广播给下一个分片，分片1中的 AggregationNode 收到所有分片0的多个实例广播的本地聚合后的数据集，做最后的数据 merge。这就比较好理解这两个分片中的 AggregationNode 的 grouping_exprs 和 aggregate_functions 中的 TExprNode 节点类型为何不同了。例如，group by a, fnv_hash(b)这个分组，分片0的 TAggregationNode 中的 grouping_exprs 中的第二个 TExpr 树描述了 fnv_hash(b)操作，由两个 TExprNode 组成，根节点类型为 FUNCTION_CALL；而分片1中 TAggregationNode 中的 grouping_exprs 中的第二个 TExpr 树只有一个 TExprNode，类型为 SLOT_REF。如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab16f2eef04.jpg" alt="图6  AggregationNode中grouping_exprs在不同执行计划分片中的表达式树实现" title="图6  AggregationNode中grouping_exprs在不同执行计划分片中的表达式树实现" />

图6  AggregationNode 中 grouping_exprs 在不同执行计划分片中的表达式树实现

#### join 操作节点 THashJoinNode

如果 TPlanNode 是一个 THashJoinNode，则有两个比较重要的字段，一个是 eq_join_conjuncts，另一个是 other_join_conjuncts。eq_join_conjuncts 的类型是 TEqJoinCondition，TEqJoinCondition 的成员是名为 left 和 right 的两个 TExpr。这个比较好理解，left 和 right 分别代表 join 子句中等号两边的表达式。例如 t1 join t2 on t1.a=t2.a，那么 left 这个 TExpr 中只有一个 TExprNode，表达式类型为 SLOT_REF，right 这个 TExpr 中也只有一个 TExprNode，表达式类型同为 SLOT_REF。这里需要重点说一下 other_join_conjuncts 这个结构。大家可以看一下 Impala 前端的 HashJoinNode 代码，其中对 other_join_conjuncts 的解释是：join conjuncts from the JOIN clause that aren't equi-join predicates，单看起来似乎说的很明确，就是在 join 子句中出现非 equi-join 的条件时会设置 other_join_conjuncts。但其实这里有个前提，就是只有在 outer join 和 semi join 这两种操作中，other_join_conjuncts 才会被设置，inner join 的情况并不会设置 other_join_conjuncts。这里通过一个例子来说明会比较容易理解。比如我有两张表，左表 t1 中的数据如下：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab12668f693.jpg" alt="" title="" />

右表 t2 中的数据如下：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab129505ef5.jpg" alt="" title="" />

对于这两张表，我们先来找出两张表姓名相同，性别不同的记录的交集，这里我们使用 inner join，SQL 语句如下：

```
select t1.name, t1.gender, t2.gender from t1 inner join t2 on t1.name = t2.name and t1.gender != t2.gender
```

在这里，impala 会自动将 t1.gender != t2.gender 转化为全局的 conjuncts，转换后的 SQL 语句为：

```
select t1.name, t1.gender, t2.gender from t1 inner join t2 on t1.name = t2.name where t1.gender != t2.gender
```

返回的结果集如下：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab13435e7c4.jpg" alt="" title="" />

这种变换是很容易理解的，先找出名字相同的记录，再全局过滤掉 join 返回的结果集中性别相同的记录。现在考虑另外一种情况，返回左表的所有记录，并以姓名相同，性别不同的条件 join 右表，这里我们使用 left outer join，SQL 语句如下：

```
select t1.name, t1.gender, t2.gender from t1 left outer join t2 on t1.name = t2.name and t1.gender != t2.gender
```

返回的结果集如下：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab13a96fd9d.jpg" alt="" title="" />

可以看到 left outer join 返回了左表的所有记录，但是由于两张表中姓名为 a 的记录的性别相同，不符合 left outer join 子句中的“姓名相同但性别不同”的约束条件，因此对应的记录中 t2.gender 的值为 NULL。那么考虑一下，可否像 inner join 那样，把 outer join 中的 non equiv-conjuncts 转化为全局的 conjuncts 呢？答案是否定的。先来看一下 left outer join 转化后的 SQL 语句和对应的结果。转化后的 SQL 语句如下：

```
select t1.name, t1.gender, t2.gender from t1 left outer join t2 on t1.name = t2.name where t1.gender != t2.gender
```

返回的结果集如下：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab1419be06c.jpg" alt="" title="" />

可以看到 outer join 子句中的 non equiv-conjuncts 转化为全局 conjuncts 之后，结果集中姓名 b 对应的记录被过滤掉了，这当然不是我们想要的结果。

之所以不能将 outer join 子句中的 non equiv-conjuncts 转化为全局 conjunts，是因为无论 join 子句中是否存在 non equiv-conjuncts，最终的结果集都应该包含左表（left outer join）或者右表（right outer join）的全部记录。因此 en_join_conjuncts 和 other_join_conjuncts 两部分信息能够决定带有 non equiv-conjuncts 的 outer join 或 semi join 的正确性，将 non equiv-conjuncts 转换成全局 conjuncts 并不是正确的做法。

#### 和查询结果列相关的表达式信息

我们提交的查询结果集中的每一列都是一个表达式，在 TPlanFragment 结构中由 output_exprs 字段表示。output_exprs 的类型为 TExpr 的集合。集合中每个 TExpr 都是一棵 TExprNode 树，包含了查询输出一列的表达式信息。例如，select abs(fnv_hash(a)), count(b) from t group by a 查询的输出有两列，第一列的 TExprNode 树的根节点为表达式类型为 FUNCTION_CALL(abs(BIGINT))，其孩子节点类型也是 FUNCTION_CALL(fnv_hash(STRING))，叶子节点类型为 SLOT_REF。

#### Output Sink

Output Sink，即数据流输出的目的地。这个目的地要么是下一个查询计划分片（select），要么是一个表（insert select 或者 create table as select）。在 TPlanFragment 结构中的 otput_sink 字段的类型是 TDataSink，TDataSink 类型的成员如下：

```
TDataSinkType::type type
TDataStreamSink stream_sink
TTableSink table_sink
```

#### TDataSinkType

根据数据流输出目的地的不同，date sink 有两种类型，一种是 DATA_STREAM_SINK，位于 data stream sender 下游查询计划分片中；一种是 TABLE_SINK，位于 coordinator 下游，是数据查询结果集和待插入的新表之间的媒介。

#### TDataStreamSink

TDataStreamSink 结构中有两个成员，一个是 dest_node_id，即目的节点的 id。例如一个select a, count(b) from t group by a 的聚合查询，执行计划最底层分片的 AggregationNode 是一个 data stream sender，它的 id 为1；它所在分片的下游分片中的 exchangeNode 是一个 data stream receiver，它的 id 为2，那么 TDataStreamSink 变量总的 dest_node_id 为2。

TDataStreamSink 的另一个成员是 TDataPartition 类型的变量 output_partition。从名字上来看，很容易让人误以为和是和表分区相关，然而并不是。TDataPartition 结构描述了数据流的分发方式，有四种分发方式，UNPARTITIONED、RANDOM、HASH_PARTITIONED 以及 RANGE_PARTITIONED。截止到 Impala2.2 版本还不支持 RANGE PARTITION 的方式。那下面我们就对 UNPARTITIONED、RANDOM和HASH_PARTITIONED 做一下解释。

1. UNPARTITIONED——顾名思义，就是“不分片”，也就是所有数据位于同一个 impalad 节点。
2. RANDOM——数据并不按照某一列分片，而是随机分布在多个节点上。例如 HdfsScanNode 的数据分片就是 RANDOM 的。
3. HASH_PARTITIONED——数据按照某一列分片，不同分片的数据位于不同的 impalad 节点。

#### TTableSink

TTableSink 结构中的结构相对简单，主要包括目的表 id、tableSink 类型（HDFS 或 HBASE）、hdfsTableSink 的 partition key 表达式，以及是否覆盖原有数据（insert overwrite）等信息。

#### Impalad 节点上的数据扫描范围

TQueryExecRequest 结构中的 per_node_scan_ranges 成员定义了和查询相关的数据扫描范围，类型是 map<TPlanNodeId, vector<TScanRangeLocations> >，为 Impalad 节点到 TScanRangeLocation 集合的映射。这个结构主要描述了需要扫描的数据在集群上的分布，包括数据位于哪些节点、每个节点上数据所在 block在文件中的偏移和大小，以及 block 的备份信息。这个结构的作用也很明显，就是根据 TScanRangeLocations 在每个 Impalad 节点上的数量（这里可以认为 TScanRangeLocations 的数量就是一个节点上需要扫描的 block 数），来决定在在一个 ScanNode 实例中，会有多少个并发的 Scanner。对于 text 文件来说，每个 Scanner 负责扫描一个 block；对于 parquet 文件来说，每个 Scanner 负责扫描一个文件。TScanRangeLocations 的成员如下：

```
TScanRange scan_range
vector<tscanrangelocation> locations</tscanrangelocation>
```

#### TScanRange

TScanRange 顾名思义，定义了一个数据分片的扫描范围。这个数据范围对于 HDFS 来说是一个 block，对于 HBASE 则是一个 key range。TScanRange 中有两个成员，一个是 THdfsFileSplit 类型的 hdfs_file_split 变量，在 THdfsFileSplit 这个结构中，定义了一个 Scanner 所需的 block 的全部信息，包括 block 所在的文件名、block 在文件中的偏移、block 的大小、block 所在的 partition id（还记得在描述符表的 Table 描述符中 THdfsTable 中定义的 partitions 成员吗？类型为 map<int64_t, THdfsPartition>，Scanner 会从这里找到 partition id 对应的 partition 信息）、文件长度以及采用的压缩算法。另一个 TScanRange 的成员是 THBaseKeyRange 类型的 hbase_key_range 变量，在 THBseKeyRange 这个结构中，存储了当前需要扫描的数据分片的起始 rowKey 和结束 rowKey。

#### TScanRangeLocation

数据分片（对 HDFS 来说是 block）的 replication 信息保存在 TScanRangeLocations 中。众所周知。HDFS 的数据默认是3备份，那么在 locations 这个集合中就存储了3个 TScanRangeLocation，每个 TScanRangeLocation 都保存了其中一个备份的相关信息，包括这个 replication 所在的主机 id、数据所在的 volumn id 以及数据是否被 hdfs 缓存。

#### Impala 的查询上下文

和用户提交的查询相关的上下文信息保存在 TQueryExecRequest 的 query_ctx 成员中，类型为 TQueryCtx。TQueryCtx 的成员如下：

```
TClientRequest request
TUniqueId query_id
TSessionState session
string now_string
int32_t pid
TNetworkAddress coord_address
vector< ::impala::TTableName>  tables_missing_stats
bool disable_spilling
TUniqueId parent_query_idx
```

#### TClientRequest

TClientRequest 结构中保存了客户端提交的 SQL 语句以及 Impala 的启动参数，包括计算节点数、scanner 一次扫描的 batch 大小、最大 scanner 线程数、最大 io 缓冲区大小、mem_limit、parquet 文件大小，以及是否启用运行时代码生成等信息。

#### TSessionState

TSessionState 结构中保存了客户端连接信息，包括客户端连接的方式（BEESWAX 或者 HIVESERVER2）、连接的数据库、用户名、以及提交查询所在节点的主机名和端口。

#### TNetworkAddress

TNetworkAddress 结构中保存了 coordinator 的主机名和端口。

### 总结

通过对 TQueryExecRequest 结构的分析，我们不仅能够了解 Impala 在一个查询的生命周期内收集了哪些有用的信息，更加重要的是，对照这些信息，能够帮助我们更好的理解 Impala 的查询执行逻辑，使得对 Impala 代码的理解更加深刻，在实际的使用场景中，根据不同的查询需求和数据量级，做出更有针对性的查询优化调整。