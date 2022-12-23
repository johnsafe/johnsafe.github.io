## 在 Apache Spark 2.0 中使用 DataFrames 和 SQL 的第一步

文/马小龙

Spark 2.0开发的一个动机是让它可以触及更广泛的受众，特别是缺乏编程技能但可能非常熟悉 SQL 的数据分析师或业务分析师。因此，Spark 2.0现在比以往更易使用。在这篇简短的文章中，我将介绍如何使用 Apache Spark 2.0。并将重点关注 DataFrames 作为新 Dataset API 的无类型版本。

到 Spark 1.3，弹性分布式数据集（Resilient Distributed Dataset，RDD）一直是 Spark 中的主要抽象。RDD API 是在 Scala 集合框架之后建模的，因此间接提供了 Hadoop Map / Reduce 熟悉的编程原语以及函数式编程（Map、Filter、Reduce）的常用编程原语。虽然 RDD API 比 Map / Reduce 范例更具表达性，但表达复杂查询仍然很繁琐，特别是对于来自典型数据分析背景的用户，他们可能熟悉 SQL，或来自 R/Python 编程语言的数据框架。

Spark 1.3引入了 DataFrames 作为 RDD 顶部的一个新抽象。DataFrame 是具有命名列的行集合，在 R 和 Python 相应包之后建模。

Spark 1.6看到了 Dataset 类作为 DataFrame 的类型化版本而引入。在 Spark 2.0中，DataFrames 实际上是 Datasets 的特殊版本，我们有 type DataFrame = Dataset [Row]，因此 DataFrame和 Dataset API 是统一的。

表面上，DataFrame 就像 SQL 表。Spark 2.0将这种关系提升到一个新水平：我们可以使用 SQL 来修改和查询 DataSets 和 DataFrames。通过限制表达数量，有助于更好地优化。数据集也与 Catalyst 优化器良好集成，大大提高了 Spark 代码的执行速度。因此，新的开发应该利用 DataFrames。

在本文中，我将重点介绍 Spark 2.0中 DataFrames 的基本用法。我将尝试强调 Dataset API 和 SQL 间的相似性，以及如何使用 SQL 和 Dataset API 互换地查询数据。借由整个代码生成和 Catalyst 优化器，两个版本将编译相同高效的代码。

代码示例以 Scala 编程语言给出。我认为这样的代码最清晰，因为 Spark 本身就是用 Scala 编写的。

### SparkSession

SparkSession 类替换了 Apache Spark 2.0中的 SparkContext 和 SQLContext，并为 Spark 集群提供了唯一的入口点。

```
val spark = SparkSession
   .builder()
   .appName("SparkTwoExample")
   .getOrCreate()
```

**代码1**

为了向后兼容，SparkSession 对象包含 SparkContext 和 SQLContext 对象，见下文。当我们使用交互式 Spark shell 时，为我们创建一个名为 spark 的 SparkSession 对象。

### 创建 DataFrames

DataFrame 是具有命名列的表。最简单的 DataFrame 是使用SparkSession 的 range 方法来创建：

```
scala> val numbers = spark.range（1，50，10）
numbers：org.apache.spark.sql.Dataset [Long] = [id：bigint]
```

**代码2**

使用 show 给我们一个 DataFrame 的表格表示，可以使用 describe 来获得数值属性概述。describe 返回一个 DataFrame：

```
scala> numbers.show()
 id
---
  1
 11
 21
 31
 41
 
scala> numbers.describe().show()
 
summary|                                  id
----------+--------------------------
      count|                                    5
      mean|                               21.0
    stddev|   15.811388300841896
         min|                                    1
        max|                                  41
```

**代码3**

观察到 Spark 为数据帧中唯一的列选择了名称 id。 对于更有趣的示例，请考虑以下数据集：

```
val customerData = List(("Alex", "浙江", 39, 230.00), ("Bob", "北京", 18, 170.00),
("Chris", "江苏", 45, 529.95), ("Dave", "北京", 25, 99.99), ("Ellie", "浙江", 23, 1299.95), ("Fred", "北京", 21, 1099.00))
val customerDF = spark.createDataFrame(customerData)
```

**代码4**

在这种情况下，customerDF 对象将有名为\_1、\_2、\_3、\_4的列，它们以某种方式违反了命名列的目的。可以通过重命名列来恢复：

```
val customerDF = spark.createDataFrame(customerData).
  withColumnRenamed("_1", "customer").
  withColumnRenamed("_2", "province").
  withColumnRenamed("_3", "age").
  withColumnRenamed("_4", "total")
```

**代码5**

使用 printSchema 和 describe 提供以下输出：

```
scala> customerDF.printSchema
root
 |-- customer: string (nullable = true)
 |-- province: string (nullable = true)
 |-- age: integer (nullable = false)
 |-- total: double (nullable = false)
 
 
scala> customerDF.describe().show
 
summary|                                age|                            total
-----------+-------------------------+------------------------
       count|                                    6|                                  6
       mean|                               28.5|   571.4816666666667
          stev|  10.876580344942981|    512.0094204374238
           min|                                 18|                            99.99
          max|                                 45|                        1299.95
```

**代码6**

一般来说我们会从文件加载数据。SparkSession 类为提供了以下方法：

```
val customerDFFromJSON = spark.read.json("customer.json")
val customerDF = spark.read.option("header", "true").option("inferSchema", "true").csv("customer.csv")
```

**代码7**

在这里我们让 Spark 从 CSV 文件的第一行提取头信息（通过设置 header 选项为 true），并使用数字类型（age 和 total）将数字列转换为相应的数据类型 inferSchema 选项。

其他可能的数据格式包括 parquet 文件和通过 JDBC 连接读取数据的可能性。

### 基本数据操作

我们现在将访问 DataFrame 中数据的基本功能，并将其与 SQL 进行比较。

#### 沿袭，操作，动作和整个阶段的代码生成

相同的谱系概念，转换操作和行动操作之间的区别适用于 Dataset 和 RDD。我们下面讨论的大多数 DataFrame 操作都会产生一个新的 DataFrame，但实际上不执行任何计算。要触发计算，必须调用行动操作之一，例如 show（将 DataFrame 的第一行作为表打印），collect （返回一个 Row 对象的 Array），count （返回 DataFrame 中的行数），foreach （对每一行应用一个函数）。这是惰性求值（lazy evaluation）的常见概念。

下面 Dataset 类的所有方法实际上依赖于所有数据集的有向非循环图（Directed Acyclic Graph，DAG），从现有数据集中创建一个新的“数据集”。这被称为数据集的沿袭。仅使用调用操作时，Catalyst 优化程序将分析沿袭中的所有转换，并生成实际代码。这被称为整阶段代码生成，并且负责 Dataset 对 RDD 的性能改进。

#### Row-行对象

Row 类在 DataFrame 的一行不带类型数据值中充当容器。通常情况下我们不会自己创建 Row 对象，而是使用下面的语法：

```
import org.apache.spark.sql._
val row = Row(12.3, false, null, "Monday")
```

**代码8**

Row 对象元素通过位置（从0开始）或者使用 apply 进行访问：

```
row(1) // 产生 Any = false
```

**代码9**

它会产生一个 Any 的对象类型。或者最好使用 get，方法之一：

```
row.getBoolean(1) // 产生 Boolean = false
row.getString(3) // 产生 String = "Monday"
```

**代码10**

因为这样就不会出现原始类型的开销。我们可以使用 isNull 方法检查行中的一个条目是否为'null'：

```
row.isNullAt(2) // 产生 true
```

**代码11**

我们现在来看看 DataFrame 类最常用的转换操作：

#### select

我们将要看的第一个转换是“select”，它允许我们对一个 DataFrame 的列进行投影和变换。

#### 引用列

通过它们的名称有两种方法来访问 DataFrame 列：可以将其引用为字符串；或者可以使用 apply 方法，col-方法或 $ 以字符串作为参数并返回一个 Column （列）对象。所以 customerDF.col("customer")和 customerDF("customer")都是 customerDF 的第一列。

#### 选择和转换列

最简单的 select 转换形式允许我们将DataFrame投影到包含较少列的 DataFrame 中。下面的四个表达式返回一个只包含 customer 和 province 列的 DataFrame：

```
customerDF.select("customer", "province")
customerDF.select($"customer", $"province")
customerDF.select(col("customer"), col("province"))
customerDF.select(customerDF("customer"), col("province"))
```

**代码12**

不能在单个 select 方法中调用混合字符串和列参数：customerDF.select("customer", $"province")导致错误。
使用 Column 类定义的运算符，可以构造复杂的列表达式：

```
customerDF.select($"customer",  ($"age" * 2) + 10, $"province" === "浙江")
```

**代码13**

应用 show 得到以下结果：

```
customer|    ((age * 2) + 10)|    (province = 浙江)
-----------+------------------+----------------------
         Alex|                     88.0|                   true
         Bob|                     46.0|                  false
       Chris|                   100.0|                  false
       Dave|                     60.0|                  false
        Ellie|                     56.0|                    true
       Fred|                     52.0|                   false
```

**代码14**

#### 列别名

新数据集的列名称从用于创建的表达式中派生而来，我们可以使用 alias 或 as 将列名更改为其他助记符：

```
customerDF.select($"customer" as "name",  ($"age" * 2) + 10 alias "newAge", $"province" === "浙江" as "isZJ")
```

**代码15**

产生与前面相同内容的 DataFrame，但使用名为 name，newAge 和 isZJ 的列。

Column 类包含用于执行基本数据分析任务的各种有效方法。我们将参考读者文档的详细信息。

最后，我们可以使用 lit 函数添加一个具有常量值的列，并使用 when 和 otherwise 重新编码列值。 例如，我们添加一个新列“ageGroup”，如果“age <20”，则为1，如果“age <30”则为2，否则为3，以及总是为“false”的列“trusted”：

```
customerDF.select($"customer", $"age",
when($"age" < 20, 1).when($"age" < 30, 2).otherwise(3) as "ageGroup", lit(false) as "trusted")
```

**代码16**

给出以下 DataFrame：

```
customer|    age|    ageGroup|    trusted
-----------+------+--------------+----------
         Alex|      39|                    3|     false
         Bob|      18|                    1|     false
       Chris|      45|                    3|     false
       Dave|      25|                    2|     false
        Ellie|      23|                    2|     false
       Fred|      21|                    2|     false
```

**代码17**

drop 是 select 相对的转换操作；它返回一个 DataFrame，其中删除了原始 DataFrame 的某些列。

最后可使用 distinct 方法返回原始 DataFrame 中唯一值的 DataFrame：

```
customerDF.select($"province").distinct
```

**代码18**

返回一个包含单个列的 DataFrame 和包含值的三行：“北京”、“江苏”、“浙江”。

#### filter

第二个 DataFrame 转换是 Filter 方法，它在 DataFrame 行中进行选择。有两个重载方法：一个接受一个 Column，另一个接受一个 SQL 表达式（一个 String）。例如，有以下两种等效方式来过滤年龄大于30岁的所有客户：

```
customerDF.filter($"age" > 30)
customerDF.filter("age > 30") //SQL
```

**代码19**

Filter 转换接受一般的布尔连接符 and （和）和 or （或）：

```
customerDF.filter($"age" <= 30 and $"province" === "浙江")
customerDF.filter("age <= 30 and province = '浙江'") //SQL
```

**代码20**

我们在 SQL 版本中使用单个等号，或者使用三等式“===”（Column 类的一个方法）。在==运算符中使用 Scala 的等于符号会导致错误。我们再次引用 Column 类文档中的有用方法。

#### 聚合（aggregation）

执行聚合是进行数据分析的最基本任务之一。例如，我们可能对每个订单的总金额感兴趣，或者更具体地，对每个省或年龄组的总金额或平均金额感兴趣。可能还有兴趣了解哪个客户的年龄组具有高于平均水平的总数。借用 SQL，我们可以使用 GROUP BY 表达式来解决这些问题。DataFrames 提供了类似的功能。可以根据一些列的值进行分组，同样，还可以使用字符串或“Column”对象来指定。

我们将使用以下 DataFrame：

```
val customerAgeGroupDF = customerDF.withColumn("agegroup",
when($"age" < 20, 1).when($"age" < 30, 2).otherwise(3))
```

**代码21**

withColumn 方法添加一个新的列或替换一个现有的列。

聚合数据分两步进行：一个调用 GroupBy 方法将特定列中相等值的行组合在一起，然后调用聚合函数，如 sum （求和值），max （最大值）或为原始 DataFrame 中每组行计算的“avg”（平均值）。从技术上来说，GroupBy 会返回一个 RelationalGroupedDataFrame 类的对象。RelationalGroupedDataFrame 包含 max、min、avg、mean 和 sum 方法，所有这些方法都对 DataFrame 的数字列执行指定操作，并且可以接受一个 String-参数来限制所操作的数字列。此外，我们有一个 count 方法计算每个组中的行数，还有一个通用的 agg 方法允许我们指定更一般的聚合函数。所有这些方法都会返回一个 DataFrame。

例如：

```
customerAgeGroupDF.
    groupBy("agegroup", "province").
    count().show()
```

**代码22**

输出以下内容：

```
agegroup|    province|    count|
-----------+------------+--------+
               2|            北京|           2|
               3|            浙江|           1|
               3|            江苏|           1|
               2|            浙江|           1|
               1|            北京|           1|
-----------+-------------+--------+
```

**代码23**

customerAgeGroupDF.groupBy("agegroup").max().show()输出：

```
agegroup|    max(age)|    max(total)|    max(agegroup)
-----------+------------+--------------+--------------------
               1|               18|             170.0|                     1
               3|               45|           529.95|                     3
               2|               25|         1299.95|                     2
```

**代码24**

最后，customerAgeGroupDF.groupBy("agegroup").min("age", "total").show()输出：

```
agegroup|    min(age)|    min(total)
-----------+------------+-------------
               1|               18|         170.0
               3|               39|         230.0
               2|               21|         99.99
```

**代码25**

还有一个通用的 agg 方法，接受复杂的列表达式。agg 在 RelationalGroupedDataFrame 和 Dataset 中都可用。后一种方法对整个数据集执行聚合。这两种方法都允许我们给出列表达式的列表：

```
customerAgeGroupDF.
  groupBy("agegroup").
  agg(sum($"total"), min($"total")).
  show()
```

**代码26**

输出：

```
agegroup|    sum(total)|    min(total)
-----------+-------------+--------------
               1|           170.0|           170.0
               3|         759.95|           230.0
               2|       2498.94|           99.99
```

**代码27**

可用的聚合函数在 org.apache.spark.sql.functions 中定义。类 RelationalGroupedDataset在Apache Spark 1.x 中被称为“GroupedData”。 RelationalGroupedDataset 的另一个特点是可以对某些列值进行透视。例如，以下内容允许我们列出每个年龄组的总数：

```
customerAgeGroupDF.
  groupBy("province").
  pivot("agegroup").
  sum("total").
  show()
```

**代码28**

给出以下输出：

```
province|          1|            2|             3
----------+-------+---------+----------
        江苏|     null|         null|    529.95
        北京|  170.0|  1198.99|         null
        浙江|     null|  1299.95|      230.0
```

**代码29**

其中 null 值表示没有省/年龄组的组合。Pivot 的重载版本接受一个值列表以进行透视。这一方面允许我们限制列数，另一方面更加有效，因为 Spark 不需要计算枢轴列中的所有值。例如：

```
customerAgeGroupDF.
  groupBy("province").
  pivot("agegroup", Seq(1, 2)).
  agg("total").
  show()
```

**代码30**

给出以下输出：

```
province|           1|              2
----------+--------+-----------
        江苏|       null|          null
        北京|    170.0|   1198.99
        浙江|       null|   1299.95
```

**代码31**

最后，使用枢纽数据也可以进行复杂聚合：

```
customerAgeGroupDF.
  groupBy("province").
  pivot("agegroup", Seq(2, 3)).
  agg(sum($"total"), min($"total")).
  filter($"province" =!= "北京"). 
  show()
```

**代码32**

输出：

```
province|2_sum(`total`)|2_min(`total`)|3_sum(`total`)|3_min(`total`)
----------+--------------+--------------+---------------+--------------
        江苏|                null|               null|             529.95|        529.95
        浙江|         1299.95|        1299.95|               230.0|         230.0
```

**代码33**

这里=!=是 Column 类的“不等于”方法。

#### 排序和限制

OrderBy 方法允许我们根据一些列对数据集的内容进行排序。和以前一样，我们可以使用 Strings 或 Column 对象来指定列：
customerDF.orderBy（"age"）和 customerDF.orderBy（$"age"）给出相同的结果。默认排序顺序为升序。如果要降序排序，可以使用 Column 类的 desc 方法或者 desc 函数：

```
customerDF.orderBy($"province", desc("age")).show()
 
customer|    province|    age|      total
-----------+------------+------+---------
        Dave|            北京|      25|      99.99
         Fred|            北京|      21|  1099.00
          Bob|            北京|      18|    170.00
        Chris|            江苏|      45|    529.95
          Alex|            浙江|     39|    230.00
          Ellie|            浙江|     23|  1299.95
```

**代码34**

观察到 desc 函数返回了一个 Column-object，任何其他列也需要被指定为 Column-对象。

最后，limit 方法返回一个包含原始 DataFrame 中第一个 n 行的 DataFrame。

### DataFrame 方法与 SQL 对比

我们已经发现，DataFrame 类的基本方法与 SQLselect 语句的部分密切相关。下表总结了这一对应关系：

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58185ae55d776.jpg" alt="" title="" />

到目前为止连接（join）在我们的讨论中已经缺失。Spark 的 DataFrame 支持连接，我们将在文章的下一部分讨论它们。
本文的第二部分将讨论完全类型化的 DataSets API，连接和用户定义的函数（UDF）。

### 使用 SQL 来处理 DataFrames

我们还在 Apache Spark 2.0中直接执行 SQL 语句。SparkSession 的 SQL 方法返回一个 DataFrame。此外，DataFrame 的 selectExp 方法也允许我们为单列指定 SQL 表达式，如下所示。为了能够引用 SQL 表达式中的 DataFrame，首先有必要将 DataFrame 注册为临时表，在 Spark 2中称为临时视图（temporary view，简称为 tempview）。DataFrame 为我们提供了以下两种方法：

- createTempView 创建一个新视图，如果具有该名称的视图已存在，则抛出一个异常；

- createOrReplaceTempView 创建一个用来替换的临时视图。
两种方法都将视图名称作为唯一参数。

```
customerDF.createTempView("customer") //register a table called 'customer'
```

**代码35**

注册表后，可以使用 SparkSession 的 SQL 方法来执行 SQL 语句：

```
spark.sql("SELECT customer, age FROM customer WHERE province = '北京'")
```

**代码36**

返回具有以下内容的 DataFrame：

```
customer|    age
-----------+------
          Bob|     18
        Dave|     25
         Fred|     21
```

**代码37**

SparkSession 类的 catalog 字段是 Catalog 类的一个对象，具有多种处理会话注册表和视图的方法。例如，Catalog 的 ListTables 方法返回一个包含所有已注册表信息的 Dataset：

```
scala> spark.catalog.listTables().show()
+----------+------------+--------------+-----------------+----------------+
|      name|    database|   description|         tableType|    isTemporary|
+----------+------------+--------------+-----------------+----------------+
|customer|          null|                   null|   TEMPORARY|                  true|
+----------+------------+--------------+-----------------+----------------+
```

**代码38**

会返回一个包含有关注册表“tableName”中列信息的 Dataset，例如：

```
spark.catalog.listColumns("customer")
```

**代码39**

此外，可以使用 DataSet 的 SelectExpr 方法执行某些产生单列的 SQL 表达式，例如：

```
customerDF.selectExpr("sum(total)") 
customerDF.selectExpr("sum(total)", "avg(age)")
```

**代码40**

这两者都产生 DataFrame 对象。

### 结束语

我们希望让读者相信，Apache Spark 2.0的统一性能够为熟悉 SQL 的分析师们提供 Spark 的学习曲线。下一篇文章将进一步介绍类型化 Dataset API 的使用、用户定义的函数以及 Datasets 间的连接。此外，我们将讨论新 Dataset API 的使用缺陷。