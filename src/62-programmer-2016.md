## 在 Apache Spark 2.0 中使用——DataFrames 和 SQL 的第二步

文/马小龙

>本文中，作者将继续使用 Apache Spark 2.0进行基本数据的分析，主题将是 Dataset/DataFrame API 和 SQL 的可交互性。

>本文第一部分使用了无类型的 DataFrame API，其中每行都表示一个 Row 对象。在本文中，我们将使用更新的 DatasetAPI。Dataset 是在 Apache Spark 1.6中引入的，并已在 Spark 2.0中使用 DataFrames 进行了统一，我们现在有了 type DataFrame = Dataset [Row]，其中方括号（[和] Scala 中的泛型类型，因此类似于 Java 的<和>）。因此，在上一篇文章中讨论的所有诸如 select、filter、groupBy、agg、orderBy、limit 等方法都以相同的方式使用。

### Datasets：返回类型信息

Spark 2.0以前的 DataFrame API 本质上是一个无类型的 API，这也就意味着在编译期间很可能会因为某些编译器错误，导致无法访问类型信息。

和之前一样，我们将在示例中使用 Scala，因为我相信 Scala 最为简洁。可能涉及的例子：spark 将表示 SparkSession 对象，代表我们的 Spark 集群。

### 例子：分析 Apache 访问日志

我们将使用 Apache 访问日志格式数据。先一起回顾 Apache 日志中的典型行，如下所示：

127.0.0.1 - - [01/Aug/1995:00:00:01 -0400] "GET /images/launch-logo.gif HTTP/1.0" 200 1839
此行包含以下部分：

- 127.0.0.1是向服务器发出请求的客户端（远程主机） IP 地址（或主机名，如果可用）；

- 输出中的第一个-表示所请求的信息（来自远程机器的用户身份）不可用；

- 输出中的第二个-表示所请求的信息（来自本地登录的用户身份）不可用；

- [01 / Aug / 1995：00：00：01 -0400]表示服务器完成处理请求的时间，格式为：[日/月/年：小时：分：秒 时区]，有三个部件："GET /images/launch-logo.gif HTTP / 1.0"；

- 请求方法（例如，GET，POST 等）；

- 端点（统一资源标识符）；

- 和客户端协议版本（'HTTP / 1.0'）。

1.200这是服务器返回客户端的状态代码。这些信息非常有价值：成功回复（从2开始的代码），重定向（从3开始的代码），客户端导致的错误（以4开头的代码），服务器错误（代码从5开始）。最后一个条目表示返回给客户端的对象大小。如果没有返回任何内容则是-或0。

首要任务是创建适当的类型来保存日志行信息，因此我们使用 Scala 的 case 类，具体如下：

```
case class ApacheLog(
   host: String, 
   user: String, 
   password: String, 
   timestamp: String, 
   method: String, 
   endpoint: String, 
   protocol: String, 
   code: Integer, 
   size: Integer
)
```

默认情况下，case 类对象不可变。通过它们的值来比较相等性，而不是通过比较对象引用。

为日志条目定义了合适的数据结构后，现在需要将表示日志条目的 String 转换为 ApacheLog 对象。我们将使用正则表达式来达到这一点，参考如下：

^(\S+) (\S+) (\S+) \[([\w:/]+\s[+\-]\d{4})\] "(\S+) (\S+)\s*(\S*)" (\d{3}) (\S+)

可以看到正则表达式包含9个捕获组，用于表示 ApacheLog 类的字段。

使用正则表达式解析访问日志时，会面临以下问题：

- 一些日志行的内容大小以-表示，我们想将它转换为0；

- 一些日志行不符合所选正则表达式给出的格式。

为了克服第二个问题，我们使用 Scala 的“Option”类型来丢弃不对的格式并进行确认。Option 也是一个泛型类型，类型 Option[ApacheLog]的对象可以有以下形式：

-None，表示不存在一个值（在其他语言中，可能使用 null）；

-Some(log)for a ApacheLog-objectlog。

以下为一行函数解析，并为不可解析的日志条目返回 None：

```
def parse_logline(line: String) : Option[ApacheLog] = {
  val apache_pattern = """^(\S+) (\S+) (\S+) \[([\w:/]+\s[+\-]\d{4})\] "(\S+) (\S+)\s*(\S*)" (\d{3}) (\S+)""".r
 
   line match {
     case apache_pattern(a, b, c, t, f, g, h, x, y) => {
         val size = if (y == "-") 0 else y.toInt
         Some(ApacheLog(a, b, c, t, f, g, h, x.toInt, size))
       }
   case _ => None
   }
}
```

最好的方法是修改正则表达式以捕获所有日志条目，但 Option 是处理一般错误或不可解析条目的常用技术。

综合起来，现在来剖析一个真正的数据集。我们将使用著名的 NASA Apache 访问日志数据集，它可以在 [ftp://ita.ee.lbl.gov/traces/NASA_access_log_Jul95.gz](ftp://ita.ee.lbl.gov/traces/NASA_access_log_Jul95.gz) 下载。

下载和解压缩文件后，首先将其打开为 String 的 Dataset，然后使用正则表达式解析：

```
import spark.implicits._
val filename = "NASA_access_log_Jul95"
val rawData = spark.read.text(filename).as[String].cache
```

用 spark.read.text 方法打开文本文件并返回一个 DataFrame，是 textfile 的行。使用 Dataset 的 as 方法将其转换为包含 Strings 的 Dataset 对象（而不是 Rows 包含字符串），并导入 spark.implicits._以允许创建一个包含字符串或其他原始类型的 Dataset。

现在可以解析数据集：

val apacheLogs = rawData.flatMap(parse_logline)

flatMap 将 parse_logline 函数应用于 rawData 的每一行，并将 Some(ApacheLog)形式的所有结果收集到 apacheLogs 中，同时丢弃所有不可解析的日志行（所有结果的形式 None）。

我们现在可以对“数据集”执行分析，就像在“DataFrame”上一样。Dataset 中的列名称只是 ApacheLog case 类的字段名称。

例如，以下代码打印生成最多404个响应的10个端点：

```
apacheLogs.filter($"code" === 404).
  groupBy($"endpoint").
  count.
  orderBy($"count".desc).
  limit(10).show
```

如前所述，可以将 Dataset 注册为临时视图，然后使用 SQL 执行查询：

```
apacheLogs.createOrReplaceTempView("apacheLogs")
 
spark.sql("select endpoint, count(*) as c
   from apacheLogs 
   where code = 404
group by endpoint 
   order by c desc 
   limit 10").show
```

上面的 SQL 查询具有与上面的 Scala 代码相同的结果。

#### 用户定义的函数（user defined function, UDF）

在 Spark SQL 中，我们可以使用范围广泛的函数，包括处理日期、基本统计和其他数学函数的函数。Spark 在函数中的构建是在 org.apache.spark.sql.functions 对象中定义的。

作为示例，我们使用以下函数提取主机名的顶级域：

```
def extractTLD(host : String) : String = {
  host.substring(host.lastIndexOf('.')  + 1)
}
```

如果想在 SQL 查询中使用这个函数，首先需要注册。这是通过 SparkSession 的 udf 对象实现的：

```
val extractTLD_UDF =spark.udf.register("extractTLD", extractTLD _)
```

函数名后的最后一个下划线将  extractTLD  转换为部分应用函数（partially applied function)，这是必要的，如果省略它会导致错误。register 方法返回一个 UserDefinedFunction 对象，可以应用于列表达式。

一旦注册，我们可以在 SQL 查询中使用 extractTLD：

```
spark.sql("select extractTLD(host) from apacheLogs")
```

要获得注册的用户定义函数概述，可以使用 spark.catalog 对象的 listFunctions 方法，该对象返回 SparkSession 定义的所有函数 DataFrame：

```
spark.catalog.listFunctions.show
```

注意 Spark SQL 遵循通常的 SQL 约定，即不区分大小写。也就是说，以下 SQL 表达式都是有效的并且彼此等价：select extractTLD（host）from apacheLogs，select extracttld（host）from apacheLogs，"select EXTRACTTLD(host) from apacheLogs"。spark.catalog.listFunctions 返回的函数名将总是小写字母。

除了在 SQL 查询中使用 UDF，我们还可以直接将它们应用到列表达式。以下表达式返回 .net 域中的所有请求：

```
apacheLogs.filter(extractTLD_UDF($"host") === "net")
```

值得注意的是，与 Spark 在诸如 filter，select 等方法中的构建相反，用户定义的函数只采用列表达式作为参数。写 extractTLD\_UDF（“host”）会导致错误。

除了在目录中注册 UDF 并用于 Column 表达式和 SQL 中，我们还可以使用 org.apache.spark.sql.functions 对象中的 udf 函数注册一个 UDF：

```
import org.apache.spark.sql.functions.udf
def request_failed(code : Integer) : Boolean = { code >= 400 }
val request_failed_udf = udf(request_failed _)
```

注册 UDF 后，可以将它应用到 Column 表达式（例如 filter 里面），如下所示：

```
apacheLogs.filter(request_failed_udf($"code")).show
```

但是不能在 SQL 查询中使用它，因为还没有通过名称注册它。

### UDF 和 Catalyst 优化器

Spark 中用 Catalyst 优化器来优化所有涉及数据集的查询，会将用户定义的函数视作黑盒。值得注意的是，当过滤器操作涉及 UDF 时，在连接之前可能不会“下推”过滤器操作。我们通过下面的例子来说明。

通常来说，不依赖 UDF 而是从内置的“Column”表达式进行组合操作可能效果更好。

### 加盟

最后，我们将讨论如何使用以下两个 Dataset 方法连接数据集：

- join 返回一个 DataFrame

- joinWith 返回一对 Datasets

以下示例连接两个表1、表2（来自维基百科）：

**表1  员工(Employee)**

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fd98295b63.jpg" alt="表1  员工(Employee)" title="表1  员工(Employee)" />

**表2  部门(Department)**

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fd98c9bef6.jpg" alt="表2  部门(Department)" title="表2  部门(Department)" />

定义两个 case 类，将两个表编码为 case 类对象的序列（由于空间原因不显示），最后创建两个 Dataset 对象：

```
case class Department (depID : Integer, depName: String)
 case class Employee  (lastname : String, depID: Integer)
val depData = Seq(Department(31, "Sales"), 
... )
 val empData = Seq(Employee("Rafferty", 31), 
 ...,
 Employee("Williams",null))
val employees = spark.createDataset(empData)
 val departments = spark.createDataset(depData)
```

为了执行内部等连接，只需提供要作为“String”连接的列名称：

val joined = employees.join(departments, "depID")

Spark 会自动删除双列，joined.show 给出以下输出：

在上面，joined 是一个 DataFrame，不再是 Dataset。连接数据集的行可以作为 Seq 列名称给出，或者可以指定要执行的 equi-join（inner，outer，left_outer，right_outer或leftsemi）类型。想要指定连接类型的话，需要使用 Seq 表示法来指定要连接的列。请注意，如果执行内部联接（例如，获取在同一部门中工作的所有员工的对）：employees.join(employees，Seq(“depID”))，我们没有办法访问连接的 DataFrame 列：employees.join(employees, Seq("depID")).select("lastname")会因为重复的列名而失败。处理这种情况的方法是重命名部分列：

```
employees.withColumnRenamed("lastname", "lname").
 join(employees, Seq("depID")).show
```

除了等连接之外，我们还可以给出更复杂的连接表达式，例如以下查询，它将所有部门连接到不知道部门 ID 且不在本部门工作的员工：

```
departments.join(employees, departments("depID") =!= employees("depID"))
```

然后可以不指定任何连接条件，在两个 Datasets 间执行笛卡尔联接： departments.join(employees).show。

**表3  输出**

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fd995e7fc7.jpg" alt="表3  输出" title="表3  输出" />

**表4  返回 Dataset**

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fd99e18a9f.jpg" alt="表4  返回Dataset" title="表4  返回Dataset" />

### 与 joinWith 类型保存连接

最后，Dataset 的 joinWith 方法返回一个 Dataset，包含原始数据集中匹配行的 Scala 元组。

```
departments.
  joinWith(employees, 
    departments("depID") === employees("depID")
).show
```

这可以用于自连接后想要规避上述不可访问列的问题情况。

### 加入和优化器

Catalyst 优化器尝试通过将“过滤器”操作向“下推”，以尽可能多地优化连接，因此它们在实际连接之前执行。

为了这个工作，用户定义的函数（UDF），不应该在连接条件内使用用因为这些被 Catalyst 处理为黑盒子。

### 结论

我们已经讨论了在 Apache Spark 2.0中使用类型化的 DatasetAPI，如何在 Apache Spark 中定义和使用用户定义的函数，以及这样做的危险。使用 UDF 可能产生的主要困难是它们会被 Catalyst 优化器视作黑盒。