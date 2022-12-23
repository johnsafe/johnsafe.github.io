## Spark MLlib 2.0 前瞻

文/梁堰波

即将发布的 Sprak2.0版本，对目前最火的分布式机器学习项目 MLlib 进行了优化。提供了新的解决回归问题的优化算法，完善机器学习，支持更多的算法、统计函数。社区还推动了一些其他相关项目。

MLlib 是 Apache Spark 项目的机器学习模块，也是目前在开源大数据界最火的分布式机器学习项目。MLlib 包含了最常用的机器学习算法以及用于特征提取、准备，模型训练、预测、评估的机器学习 Pipeline。本文主要讨论在即将发布的 Spark 2.0中，MLlib 主要的一些变化。

### 最优化算法

最优化算法是许多机器学习算法的基础，常见的最优化算法有梯度下降/随机梯度下降、牛顿法等：

- 梯度下降算法收敛速度较慢。随机梯度下降算法需要在每一步迭代过程中采样出 mini batch 比例的数据，但是目前 Spark 还没有一个非常高效的采样算法，导致算法的效率不高。所以目前在 Spark ML 中很少采用梯度下降/随机梯度下降算法。

- 牛顿法的收敛速度会比梯度下降算法快，但是传统牛顿法在计算过程中需要计算和保存目标函数的二阶偏导数，计算复杂度和存储成本高。基于这些原因发展出了拟牛顿法，L-BFGS 就是拟牛顿法的一种。L-BFGS 使用一阶导数去近似 Hessian 矩阵的逆，降低了存储成本和计算复杂度。同时对于 L1 regularization 问题，可以使用 OWL-QN 算法来解决。

目前在 Spark ML 里面所有的凸优化问题默认都是使用 L-BFGS/OWL-QN 算法。

Spark 2.0提供了用于解决回归问题的 Iteratively reweighted least squares （以下简称 IRLS）优化算法。IRLS 算法主要用于解决以下面的形式为目标函数的优化问题：

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb6fb3e4ffa.jpg" alt="" title="" />

问题求解通过迭代的方式，迭代过程中每一步实际上是解一个 Weighted Least Squares （以下简称 WLS）问题：

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb6ff0d5d75.jpg" alt="" title="" />

WLS 算法是 Spark 1.6开始提供的一个线性回归问题算法，通过normal equation 思路求解。这个算法的特点是只需要把数据读入构造一个矩阵方程，然后通过 Cholesky 分解的方式解这个方程即可得到最后的模型。因为不需要迭代求解最优化问题，整个计算的效率会提高很多。但是 Cholesky 分解是在 Driver 上进行的，所以数据集的 feature 维度不能太大(<=4096)，否则会影响性能。

IRLS 优化算法在 Spark 里面主要用于解决 Generalized Linear Models （以下简称 GLM）类算法的最大似然估计问题，未来也可以解决 Robust Regression 等问题。由于 IRLS 算法的每一步迭代过程是解决一个 WLS 问题，所以 IRLS 算法同样限制在 feature 维度不大于4096的条件下，但是对于训练样本数目是没有限制的。IRLS 算法的这个限制对于大多数传统行业的应用足够了，但可能在互联网行业应用中会受到一些限制，所以对于 feature 维度大于 4096 的条件下，可以继续使用 L-BFGS 优化算法解决。当 feature 维度非常大（例如大于10亿）的时候，我们需要模型的并行化，社区也在研究使用Vector-free L-BFGS（http://papers.nips.cc/paper/5333-large-scale-l-bfgs-using-mapreduce.pdf）来解决这类问题，不过在这个版本中不会支持。

### 算法

算法是一个机器学习项目最重要的特征，表1列出了到2.0的时候 Spark MLlib 支持的机器学习算法。

**表1  2.0版本支持机器学习算法**

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb82092ae56.jpg" alt="表1  2.0版本支持机器学习算法" title="表1  2.0版本支持机器学习算法" /> 

除了表中所列算法以外，还有用于频繁项挖掘的算法（FPGrowth 和 PrefixSpan）。

其中在即将发布的 Spark 2.0中，最大的改动是增加了 Generalized Linear Models。GLM 指的是统计学中的广义线性模型，是对传统 Linear Regression 的扩展，指定不同的 family 和 link 可以得到不同的模型。传统的 Linear Regression 因变量的误差分布是正态分布（高斯分布），GLM 的误差分布可以是指数分布簇里的其他分布：二项分布、泊松分布、伽马分布等。GLM 的应用非常广泛，在金融、电信、交通、医学和科学研究等多个领域都有非常重要的作用。
GLM 一般由三部分组成：

- 因变量的误差分布，用 family 表示。

- 线性预测器：

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb717972b12.jpg" alt="" title="" />

- 用于表示线性预测器与因变量均值之间关系的link函数：

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb71c0e8744.jpg" alt="" title="" />

Spark ML 里面的 GLM 算法实现是 Generalized-LinearRegression 类，用户可以通过 Scala/Java/Python 和 R 接口使用，GLM 的输出结果可以匹配 R glm 的输出。Spark 自带了四个 family，每个 family 支持3种 link 函数，这样就可以演变出12种不同的 GLM 模型： 

**表2 GLM 模型**

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb71f38db14.jpg" alt="表2 GLM模型" title="表2 GLM模型" />

在 Spark 2.0中，GLM 算法只提供了 IRLS 解法，也就是限定 feature 维度不大于4096的条件下，未来会加入 L-BFGS 解法的支持。我们都知道 Linear Regression 和 Logistic Regression 问题相当于 GLM 算法在因变量误差分布是正态分布和二项分布情况下的特例，所以当 feature 维度大于4096的时候，可以直接调用 Spark ML 的 LinearRegression 和 LogisticRegression 实现，这两个模型都支持 L-BFGS 解法。Spark 在未来的版本中会考虑把这些模型统一接入 GLM 接口。

下面详细说说 GLM 的这两种解法：

- IRLS 解法：和 R glm 的解法基本一致，输出结果也是匹配的。迭代优化过程的每一步通过 reweight 函数计算出每个样本新的 weight 和 offset，然后解这个 WLS 问题。这样迭代下去直至收敛。这种解法能够输出模型的统计信息，例如 residuals、coefficients standard errors、p-value、 t-value、dispersion parameter、null deviance、residual deviance、degree of freedom、aic 等，这些统计指标可以帮助用户理解模型。这种解法最大的优势在于可以使用较少的迭代次数训练出模型。

- L-BFGS 解法：和 R glmnet 的解法类似。这种解法在机器学习领域更常见，指定这个优化问题的损失函数和梯度，然后通过求极值的方法来求解。这种解法目前还不能提供上述各种统计指标。目前 Spark GLM 接口还不支持直接调用这种解法，用户需要显式调用对应的模型（例如 LogisticRegression）。

### SparkR 接口

R 语言应用广泛，同时提供各种算法库用于机器学习。Spark 2.0将会支持更多的基于 MLlib 的分布式的机器学习算法，包括 glm、survreg、kmeans、 naiveBayes 等。

- SparkR glm 实现了 R glm 的核心功能，支持基本的 R formula （.', '~', ':', '+'和 '-'）用于指定一个 glm 模型的 feature 和 label。glm 输出的模型可以调用 summary 得到这个模型的统计信息，帮助用户调优模型。模型训练、预测和统计信息输出都是与 R glm 匹配的。在数据量增大单机无法处理的时候，用户可以直接把以前在单机上用 R glm 训练的模型移植到 Spark 平台上训练。

- Spark 2.0也提供了 MLlib 中 Survival Regression 的 R 接口survreg。Survival Regression 是一个在生物学、医学、保险学、可靠性工程学、社会学等方面都有重要应用的模型，用于对有截尾数据（censored data）进行回归分析。SparkR 的生存回归算法训练出的模型匹配 R survreg，同时也支持基本的 R formula。

### 机器学习 Pipeline

Spark 2.0的一个重要目标就是完善机器学习 Pipeline。

- 最大的变化就是合并 Estimator 和 Model 两个组件了，这是一个break change。现在的 ML 里面的算法和模型是分开的，例如LinearRegression（Estimator）和 LinearRegressionModel（Model）。在 Spark 2.0之后算法和模型会合并成一个类，让用户使用起来更方便，也减少了代码的冗余，同时这个架构与 sklearn 也是一致的。

- 由于 Python 在机器学习工作中使用率非常高，所以 Spark MLlib 2.0的目标是 Python 的 API 与 Scala 的完全对等。其中最重要的一件事就是 Pipeline 和模型的持久化。在实际应用中用户需要把训练好的模型存储到文件系统中，然后应用方直接从文件系统中读取这个模型，进行预测。从 Spark 2.0开始，所有的 Estimator 和 Transformer 都会支持 Python API 的 read/write 操作。

- 另外一个工作就是把向量、矩阵以及线性代数运算库分离出来一个单独的模块，这样能够降低代码依赖的复杂度，简化 Spark MLlib 模型的线上部署工作。例如在用于预测的生产服务器上不需要部署整个 Spark 及其依赖库，只需要部署这个单独的模块和 MLlib 模块即可。

### 统计函数

在单机时代很多统计函数一旦到了数据量比较大、分布式存储的时代实现起来就相对比较困难，有些会有一些近似的算法。Spark 2.0会支持更多的分布式统计函数，例如用于计数的 Count-Min Sketch 算法、近似分位数和中位数的算法（Greenwald-Khanna 算法的一个变种）、用于查询某个元素在集合中存在与否的 Bloom filter 算法等。这些算法可以直接调用 DataFrame 的 API 使用。

### GraphFrames 和 spark-sklearn

除了 Spark 项目里的一些变化，社区也在推动一些相关的项目，这里面最值得关注的就是 GraphFrames 和 spark-sklearn 了。

Spark 项目的 GraphX 模块是构建在 RDD 之上的图计算引擎。随着Spark的核心转向 DataFrame 和 Dataset，社区推出了 GraphFrames 这个基于 DataFrame 构建的图计算引擎。虽然目前 GraphFrames 还是一个单独的开源项目，但是我猜测未来这个会被合并到 Spark 成为新的图计算引擎。GraphFrames 的主要优点有：

- 用户可以使用类似 SQL 和 DataFrame 操作的 API 去做图查询。

- 图数据的存储和读取操作可以利用 DataFrame 所支持的所有的 data source。

- GraphFrames 支持所有的 GraphX 的算法（例如 PageRank、SVD++等），还支持像 BFS、motif finding 等新的算法。

- GraphFrames 与 GraphX 的数据可以实现无缝转换。

- GraphFrames 从诞生之日起就支持 Scala/Java/Python 三种接口。很多还在抱怨 Spark GraphX 至今没有 Python 接口的朋友可尝试下 GraphFrames。

scikit-learn （以下简称 sklearn）是单机时代一个应用非常广泛的机器学习平台，同时提供了丰富的机器学习算法库。我们能不能把 Spark 的分布式特性与 sklearn 的算法丰富性结合起来呢？spark-sklearn 就是为了这个目的而服务的一个项目。例如我们经常会用 sklearn 同时去训练多个模型，然后选择最好的使用，这就是典型的 cross vlidation 的过程。spark-sklearn 提供了一个并行 cross-validator 类，与 sklearn 的 cross-validator 是完全兼容的。这样用户只需要指定调用 spark-sklearn 的 cross-validator 就可以利用整个集群的分布式资源用来并行训练和选择模型。实验结果表明，在分布式环境下模型的训练效率会随着集群规模的增长而线性提升。

Spark MLlib 2.0除了增加了 GLM 这个算法以外，其他更多的是算法和功能的增强，以及机器学习生态系统的构建。Spark MLlib 与其他机器学习平台相比有着更强烈的统计色彩，也与 DataFrame 有着更紧密的关系，所以应用场景也超越了传统的机器学习的任务。希望有更多的人使用和贡献，让 Spark MLlib 越来越稳定、性能越来越高、有更多的新特性加入。