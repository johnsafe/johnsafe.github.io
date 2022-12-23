## CaffeOnSpark 解决了三大问题



文/周建丁

>Andy Feng 是 Apache Storm 的 Committer，同时也是雅虎公司负责大数据与机器学习平台的副总裁。他带领雅虎机器学习团队基于开源的 Spark 和 Caffe 开发了深度学习框架 CaffeOnSpark，以支持雅虎的业务团队在 Hadoop 和 Spark 集群上无缝地完成大数据处理、传统机器学习和深度学习任务，并在 CaffeOnSpark 较为成熟之后将其开源（<https://github.com/yahoo/CaffeOnSpark>）。Andy Feng 接受《程序员》记者专访，从研发初衷、设计思想、技术架构、实现和应用情况等角度对 CaffeOnSpark 进行了解读。

### CaffeOnSpark 概况
CaffeOnSpark 研发的背景，是雅虎内部具有大规模支持 YARN 的 Hadoop 和 Spark 集群用于大数据存储和处理，包括特征工程与传统机器学习（如雅虎自己开发的词嵌入、逻辑回归等算法），同时雅虎的很多团队也在使用 Caffe 支持大规模深度学习工作。目前的深度学习框架基本都只专注于深度学习，但深度学习需要大量的数据，所以雅虎希望深度学习框架能够和大数据平台结合在一起，以减少大数据/深度学习平台的系统和流程的复杂性，也减少多个集群之间不必要的数据传输带来的性能瓶颈和低效（如图1）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb87377a35a.jpg" alt="图1  深度学习集群与大数据集群分离的低效" title="图1  深度学习集群与大数据集群分离的低效" />

图1 深度学习集群与大数据集群分离的低效

CaffeOnSpark 就是雅虎的尝试。对雅虎而言，Caffe 与 Spark 的集成，让各种机器学习管道集中在同一个集群中，深度学习训练和测试能被嵌入到 Spark 应用程序，还可以通过 YARN 来优化深度学习资源的调度。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb8746aa40f.jpg" alt="图2  CaffeOnSpark系统架构" title="图2  CaffeOnSpark系统架构" />

图2 CaffeOnSpark 系统架构

CaffeOnSpark 架构如图2所示，Spark on YARN 加载了一些执行器（用户可以指定 Spark 执行器的个数（–num-executors <# of EXECUTORS>），以及为每个执行器分配的 GPU 的个数（-devices <# of GPUs PER EXECUTOR>））（Executor）。每个执行器分配到一个基于 HDFS 的训练数据分区，然后开启多个基于 Caffe 的训练线程。每个训练线程由一个特定的 GPU 处理。使用反向传播算法处理完一批训练样本后，这些训练线程之间交换模型参数的梯度值，在多台服务器的 GPU 之间以 MPI Allreduce 形式进行交换，支持 TCP/以太网或者 RDMA/Infiniband。相比 Caffe，经过增强的 CaffeOnSpark 可以支持在一台服务器上使用多个 GPU，深度学习模型同步受益于 RDMA。

考虑到大数据深度学习往往需要漫长的训练时间，CaffeOnSpark 还支持定期快照训练状态，以便训练任务在系统出现故障后能够恢复到之前的状态，不必从头开始重新训练。从第一次发布系统架构到宣布开源，时间间隔大约为半年，主要就是为了解决一些企业级的需求。

### CaffeOnSpark 解决了三大问题
《程序员》：在众多的深度学习框架中，为什么选择了 Caffe？

Andy Feng：Caffe 是雅虎所使用的主要深度学习平台之一。早在几个季度之前，开发人员就已将 Caffe 部署到产品上（见 Pierre Garrigues 在 RE.WORK 的演讲），最近，我们看到雅虎越来越多的团队使用 Caffe 进行深度学习研究。作为平台组，我们希望公司的其它小组能够更方便地使用 Caffe。

在社区里，Caffe 以图像深度学习方面的高级特性而闻名。但在雅虎，我们也发现很容易将 Caffe 扩展到非图像的应用场景中，如自然语言处理等。

作为一款开源软件，Caffe 拥有活跃的社区。雅虎也积极与伯克利 Caffe 团队和开发者、用户社区合作（包括学术和产业）。

《程序员》：除了贡献到社区的特性，雅虎使用的 Caffe 相对于伯克利版本还有什么重要的不同？

Andy Feng：CaffeOnSpark 是伯克利 Caffe 的分布式版本。我们对 Caffe 核心只做了细微改动，重点主要放在分布式学习上。在 Caffe 的核心层面，我们改进 Caffe 来支持多 GPU、多线程执行，并引入了新的数据层来处理大规模数据。这些核心改进已经加入了伯克利 Caffe 的代码库，整个 Caffe 社区都能因此而受益。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb874fc6e2f.jpg" alt="图3  CaffeOnSpark支持由网络连接的GPU和CPU集群" title="图3  CaffeOnSpark支持由网络连接的GPU和CPU集群" />

图3  CaffeOnSpark 支持由网络连接的 GPU 和 CPU 集群

如图3所示，CaffeOnSpark 支持在由网络连接的 GPU 和 CPU 服务器集群上进行分布式深度学习（用户可以自行选择向成本还是性能倾斜）。只需稍微调整网络配置，就能在自己的数据集上进行分布式学习。

《程序员》：CaffeOnSpark 希望填补 Spark 在分布式深度学习方面的空白，但 Caffe 设计的初衷是解决图像领域的应用问题，而深度学习的应用不止于图像，能否介绍 CaffeOnSpark 目前具体支持的算法，以及使用场景有哪些？

Andy Feng：正如之前所提到的，Caffe 在深度学习的应用远不止图像。举个例子，Caffe 可以为搜索引擎训练 Page Ranking 模型。在那种场景下，训练数据由用户的搜索会话组成，包括搜索词、网页的 URL 和内容。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb8768dd16d.jpg" alt="图4  雅虎希望CaffeOnSpark成为Spark上的深度学习包" title="图4  雅虎希望CaffeOnSpark成为Spark上的深度学习包" />

图4  雅虎希望 CaffeOnSpark 成为 Spark 上的深度学习包

《程序员》：Caffe 在与 Spark 的集成需要解决哪些问题？您的团队克服了哪些主要挑战？

Andy Feng：我们主要做了三个方面的事情：

**性能**

CaffeOnSpark 是按照专业深度学习集群的性能标准而设计。

典型的 Spark 应用分配多个执行器（Executor）来完成独立的学习任务，并且用 Spark 驱动器（Driver）同步模型更新。二级缓存之间是相互独立的，没有通信。这样的结构需要模型数据在执行器的 GPU 和 CPU 之间来回传输，驱动器成了通讯的瓶颈。

为了实现性能标准，CaffeOnSpark 在 Spark 执行器之间使用了 peer-to-peer 的通讯模式。最开始我们尝试了开源的 OpenMPI，但 OpenMPI 的应用需要提前选好一些机器构建 MPI 集群，而 CaffeOnSpark 其实并不知道会用到哪些机器，所以我们研发了自己的 MPI。

如果有无限的带宽连接，这些执行器就能够直接读取远端执行器的 GPU 内存。我们的 MPI 实现将计算和通讯的消耗分布到各个执行器上，因此消除了潜在的瓶颈。

**大数据**

Caffe 最初的设计只考虑了单台服务器，也就是说输入数据只在本地文件系统上。对于 CaffeOnSpark，我们的目的是处理存储在分布式集群上的大规模数据，并且支持用户使用已经存在的数据集（比如 LMDB 数据集）。CaffeOnSpark 引入了 Data Source 的概念，提供了 LMDB 的植入实现、数据框架、LMDB 和序列文件。CaffeOnSpark 不仅提供了深度学习的高级 API，也提供了非深度学习和普通数据分析/处理的接口。详见我们博文的介绍。

**编程语言**

尽管 Caffe 是用 C++ 实现的，但作为 Spark 的产品，CaffeOnSpark 也支持 Scala、Python 和 Java 的可编程接口。内存管理对在 JVM 上长期运行的 Caffe 任务是一个挑战，因为 Java 的 GC 机制并没有考虑 JNI 层的内存分配问题。我们在内存管理上做了重大改变，通过一个自定义 JNI 实现。

《程序员》：目前 Spark + MPI 架构相比深度学习专用 HPC 集群的性能差异如何？TCP 方法和 RDMA 方法的差别又是多大？

Andy Feng：我们的设计和 Spark + MPI 实现旨在取得和 HPC 集群相似的性能。这也是我们支持诸如无限带宽连接接口的原因。关于差别，我们还需要做一些扩展性的对照测试。

《程序员》：关于并行训练中常用的参数服务器（Parameter Server），CaffeOnSpark 是如何设计的？

Andy Feng：参数服务器的主要目的是可以建立很大的模型，但目前的深度学习模型还很小，所以参数服务器的用处还不是很大。当然，我们也提供了参数服务器的功能，在需要的时候启用即可。

《程序员》：另一款 Spark 深度学习框架是伯克利 AMPLab 出品的
SparkNet（<https://github.com/amplab/SparkNet>），此外《程序员》曾刊载过百度团队研发的 Spark on PADDLE（<http://geek.csdn.net/news/detail/58867>），也是针对 HDFS 数据传输的瓶颈做了优化。您如何看待这些框架的不同？

Andy Feng：CaffeOnSpark 和 SparkNet 都是为了在 Spark 集群上运行基于 Caffe 的深度学习项目。SparkNet 用标准的 Spark 结构，在 Spark 执行器内用 Spark 驱动器和 Caffe 处理器通讯。

然而，CaffeOnSpark 在 Caffe 处理器之前使用 peer-to-peer 通讯（MPI），避免驱动器潜在的扩展瓶颈。CaffeOnSpark 支持 Caffe 用户已有数据集（LMDB）和配置、增量学习、以及机器学习管道的高级接口。详情参见 <https://github.com/yahoo/CaffeOnSpark#why-caffeonspark>。

对于 Spark on PADDLE，我简单地浏览了《程序员》的文章，它的框架结构（和 CaffeOnSpark）非常相似。由于 Spark on PADDLE 项目没有公开，不清楚它是怎么和 Spark 生态系统整合的，比如 MLlib 或者 Spark SQL。

《程序员》：能否介绍 CaffeOnSpark 社区的规划，对开发者有什么要求？

Andy Feng：我们正在组织一个活跃的 CaffeOnSpark 社区。我们欢迎社区成员尝试使用 CaffeOnSpark、提出问题、查找缺陷以及贡献解决方案。我们会采用 GitHub 审阅流程来确保所有代码都是高质量的。

《程序员》：能否介绍 CaffeOnSpark 开源之后的应用反馈？接下来它的功能改进和问题解决重点包括哪些？

Andy Feng：目前 CaffeOnSpark 支持同步分布式学习，我们计划在不久的将来支持异步分布式学习。我们的 MPI 也能通过改进来处理系统崩溃问题。

如我们预计，CaffeOnSpark 开源之后，每天的访问量都在增长，包括一些国内的用户。很多人为他们在使用过程中遇到的一些问题向我们寻求帮助（估计用于图像类的应用比较多）。

例如，CaffeOnSpark 发布的 API 主要是 Scala API，一些用户提出了 Java API 的需求。这个请求是合理的，因为 Spark 支持 Scala、Python 和 Java 三种语言的 API。我们会根据需求的强烈程度规划改进，但预计 Python API 很快就会出来。另外一个是 UI 方面的问题，我们后续也会逐步改进。

同时，用户之间也在交流他们使用的经验。因为雅虎能够自己测试的实际环境比较有限，针对一些问题我们也只能给出建议，用户相互的交流会找到一些方案，让 CaffeOnSpark 的发展更好。

《程序员》：您对 CaffeOnSpark 在 Spark 社区的发展有什么样的期待？

Andy Feng：我们希望 CaffeOnSpark 促使 Spark 社区致力于深度学习领域。在雅虎，我们已经看到深度学习在大数据集上取得的显著成果，也期待着听到社区成员传来更多的好消息。