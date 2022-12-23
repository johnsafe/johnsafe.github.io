## 层次化存储：以高性价比终结 Spark 的 I/O 瓶颈

文/俞育才

NVMe SSD 是由英特尔、三星、SanDisk、Dell 等多家巨头联合推进的新一代 SSD 行业标准。它的读写性能数倍于 SATA SSD，随机访问更是机械硬盘的千倍。英特尔的 Spark 团队，重构了 Apache Spark 文件分配模块代码，合理地搭配 NVMe SSD 和普通机械硬盘，设计出一种多层级的存储结构，不仅彻底终结了 Spark 的 I/O 瓶颈，使应用的性能显著提高，在性价比上也非常具有优势。

### NVMe SSD 介绍 

与以前的 SSD 相比，NVMe SSD 不再通过 SATA 线连接，而是直接连到了 PCIe 总线上，接口协议也采用全新的、专为 SSD 设计的 NVMe 协议。这种改动给它带来了很多优良的特性，比如：高带宽、低延时、高 IOPS、低功耗等。在本文，我们主要关注其中的两个特性：高带宽和高 IOPS。一块 NVMe SSD 的带宽是2.8GB/秒，IOPS 可以到46万，这相当于7块 SATA SSD 通过 HBA 做聚合。而和机械硬盘相比，一块硬盘的带宽是100 MB/秒，NVMe SSD 是它的28倍。机械硬盘还有个问题，就是 IOPS 不高，一般也就几百，最直接的反应就是随机访问性能差。NVMe SSD 的 IOPS 是46万，这是它的千倍。所以，简单地说来，一块 NVMe SSD 可以取代7块 SATA SSD，带宽是普通硬盘28倍，IOPS 可以达到千倍。

NVMe SSD 有两种连接方式（如图1所示）：一种是插卡式，安装方式类似显卡，另一种方式可以支持热插拔。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1bd476b87.jpg" alt="图 1  NVMe SSD" title="图 1  NVMe SSD" />

**图 1  NVMe SSD**

### 使用 NVMe SSD 加速大数据处理 

NVMe SSD 在性能上完爆硬盘，但价格也相对较贵。该如何在 Spark 中应用呢？一个很直接的想法是搭配它和普通硬盘做层次化存储：用 SSD 来优先缓存需要放到外存上的数据，这主要包括 Shuffle 的数据和从内存中 evict 出来的 RDD 等，当 SSD 的容量不够时，再使用机械硬盘的空间。通常情况下，放到外存上的数据不会超过 NVMe SSD 的容量，即便超过了，由于会写到机械硬盘里，也不会造成数据丢失出错，只是性能会略微下降。

业界也早已有层次化存储的探索，比如 Tachyon （现在已更名为 Alluxio）。但是，对于 Spark 应用而言，集成 Tachyon 作层次化还有几个明显的缺点：

- 只能缓存 RDD 型的数据，不能缓存 Shuffle 的数据。

- 复杂化了软件堆栈：它引入了额外的软件组件，额外的部署以及维护的成本。

- 不可避免的性能损失：包括运行 Tachyon daemon 以及进程通信带来的额外负担。

我们能不能在 Spark 中原生支持层次化存储？

在 Spark 中，那些从内存中 evict 出的 RDD 数据、外部排序数据、以及需要 Shuffle 的数据等都是需要放在外存中的，通常我们会为它们配置临时文件夹，均匀地分配在若干个磁盘上，Spark 还无法根据磁盘的带宽、IOPS 等特性区别对待。我们修改了这部分逻辑（详见https://github.com/apache/spark/pull/10225），使 Spark 具备了识别上述层次化设备的能力，从而优先利用 SSD 缓存这些数据。
使用起来也很简单，只需要做两件事：

- 定义存储设备的优先级和阈值，在 spark-default.xml 中：

  <img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1c6f0828d.jpg" alt="" title="" />

  这表示第一级的存储层次是 SSD，那么除去 SSD，剩下的存储器就成为了后备存储，一般来说，就是系统里面的所有的 HDD。30 GB 则是 SSD 这个存储层次的阈值，也就是说，当 SSD 的容量小于30 GB 时，开始使用下一级别的存储。

- 配置 SSD 的位置，这也很容易，将“SSD”作为关键字放到 local  dir 中，如果使用的是 YARN，那么在 yarn-site.xml 中修改如下的配置。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1d1f56b5e.jpg" alt="       图2  yarn-site.xml" title="       图2  yarn-site.xml" />

**图2  yarn-site.xml**

在这两步完成以后，当 Spark 被启动时，它就会识别出这个层次化的存储配置。在应用需要磁盘空间的时候，Spark 优先放在 SSD 中，当 SSD 的容量小于30 GB 时，再放到机械硬盘中。

我们用一个来自客户的机器学习案例（NWeight）来评估这个改动。NWeight 可以计算图里任意 n-hop 远的两个节点之间的相关性，常被用做朋友之间的推荐，以及视频相关度分析等。在实现上，它的算法类似Page Rank，是一种 Spark 上的图计算。测试结果如图3所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1d50527b2.jpg" alt="图3  1 NVMe + HDDs Hierarchy Store" title="图3  1 NVMe + HDDs Hierarchy Store" />

**图3  1 NVMe + HDDs Hierarchy Store**

- 纯 SSD 场景：1块 NVMe SSD 的效果甚至和11 SATA SSD 的性能是一样的，这是因为在大数据处理的场景中，SSD 将瓶颈都推到了 CPU 上。

- 层次化存储方案：
 1. 没有额外的开销：最好的情况下和纯 SSD（NVMe/SATA SSD）一样，最坏情况和纯机械硬盘一样。

2. 和11块硬盘相比，至少会有1.82倍的性能提升（CPU 成为了瓶颈）。

3. 和 Tachyon 相比，仍然显示出1.3倍的性能提升，这主要得益于我们可以同时缓存 RDD 和 Shuffle，并且省去了大量的进程间通信。

### SSD 加速原因分析 

回到 NWeight 的例子，每个 Stage 的执行信息都打印出来得到图4这样的表格。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1e1a6da83.jpg" alt="图4  NWeight Stage 执行信息" title="图4  NWeight Stage 执行信息" />

**图4  NWeight Stage 执行信息**

注意到一个很有意思的现象，在大部分 Stage，SSD 的性能提升并不特别大，真正拉开差距的是 Stage 11和 Stage 17，这两个 Stage 有何不同？

通过分析源代码，我们发现在 NWeight 里，包含了5个主要 I/O 模式：Map 阶段的 RDD Read 和 Shuffle Write，以及 Reduce 阶段的 RDD Read，RDD Write 和 Shuffle Read，如图5所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1e83b6a76.png" alt="图5  5种IO模式" title="图5  5种IO模式" />

**图5  5种 IO 模式**

blktrace 是 Linux 内核提供的工具，通过它可以捕获操作系统发向磁盘的每个 I/O，图6是一段 blktrace 的信息。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1f00c9a35.png" alt="图6  blktrace 片断" title="图6  blktrace 片断" />

**图6  blktrace 片断**

红色框显示了一个写 I/O 的过程，在43.740s这个时间，它向设备发出一个写请求（D），要求从地址52090704开始读取560个 sector，到43.748s，这个 I/O 被完成（C）。类似的，蓝色是一个读 I/O 的过程。

通过解析诸如上面的原始信息，可以为每一种模式生成各种 I/O 图表，比如 I/O Size 直方图，Latency 直方图，寻址偏差直方图和 LBA 时间线等等，然后从这些图表中，我们就可以去理解 I/O 的特性。

在分析了大量数据后，我们得出了两个结论：

- RDD Read/Write，Shuffle Write 都是顺序的。

- Shuffle Read 是强随机的。

Stage 11和 Stage 17都包含了这个 Shuffle Read 过程，那么 Shuffle Read 带来的随机读对性能影响到底有多严重？

为了评估这个问题，我们又做了下面这个实验。在一个4节点的集群中，用11块工业级的高端机械硬盘和1块 Intel P3600（NVMe SSD）做对比，通过修改 Spark 源代码，让 Shuffle Read 分别发生在那11块机械硬盘或者 Intel P3600上，跟踪 I/O 数据，做性能分析。
首先，来看带宽。如图7所示，在做 Shuffle Read 的时候，每块磁盘的带宽仅能达到40 MB （正常情况下，一块硬盘的带宽是100 MB），并且注意到 CPU 出现较多的 I/O 等待，这意味着磁盘已经满荷，I/O 达到瓶颈，带宽确实上不去了。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1f9cec56c.png" alt="图7  磁盘在Shuffle Read时的带宽瓶颈" title="图7  磁盘在Shuffle Read时的带宽瓶颈" />

**图7  磁盘在 Shuffle Read 时的带宽瓶颈**

如果换成 SSD 呢，来看图8，绿色是11块机械硬盘合起来的带宽，蓝色是 Intel P3600的带宽，实测的 Intel P3600带宽是机械硬盘的2倍。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa1fd157552.png" alt="图8  磁盘与SSD Shuffle Read带宽对比" title="图8  磁盘与SSD Shuffle Read带宽对比" />

**图8  磁盘与 SSD Shuffle Read 带宽对比**

这从执行的时间上也可以很好地反映，见图9，SSD 的执行时间几乎是 HDD 的一半。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa200fdd637.png" alt="图9  磁盘与SSD Shuffle Read执行时间对比" title="图9  磁盘与SSD Shuffle Read执行时间对比" />

**图9  磁盘与 SSD Shuffle Read 执行时间对比**

注意，我们现在看到的还只是单一个 Shuffle Read 造成的影响，在真实的 Shuffle Stage 中，伴随着 Shuffle Read，往往还同时发生了数据 RDD Read/Write，但这时候，机械硬盘因为已经到瓶颈了（40 MB），无法输出更高的带宽，性能被定格。而在 SSD 的情况下，就可以轻松输出更高的带宽，所以实际的情况下 Shuffle Stage 的提升达到了前面说的3倍。

Shuffle Read 因为与生俱来的随机性，成为了 Spark 中 I/O 瓶颈的最重要成因。为了缓解 I/O，可以在系统里面加入更多的硬盘。但是又因为随机访问一直是传统机械硬盘的弱项，所以如果只是单纯的加入普通机械硬盘只能事倍功半。那么，我们通过加入一块 NVMe SSD 做运行时缓存，凭借着 SSD 优秀的随机访问能力，Shuffle 阶段很容易就有好几倍的提升。

这里要更正一个误区，并不是只有随机访问会导致 I/O 瓶颈，大到一定量的顺序读写一样会导致 I/O 瓶颈。比如，在机器学习算法里面，经常需要反复地迭代数据，虽然对数据的读写是顺序的，但是因为数据量大，有的时候 I/O 瓶颈也很明显。只是和随机访问相比，顺序访问需要的量要大得多。而一旦出现了这样的情况，使用 SSD 也会有比较明显的提升。所以从这个角度讲，我们把 RDD 数据和 Shuffle 数据做统一的层次化处理也是非常合理的一个做法。

使用 NVMe SSD 后，整个集群的瓶颈完全推到了 CPU 上，I/O 方面已经基本没有问题了。

如果 CPU 的性能更强悍，使用 SSD 和机械硬盘的差距会更加大。我们把 CPU 换成了 Intel Haswell 之后 Shuffle Stage 的提升可以达到3-5倍，端到端的性能提升接近3倍！

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fa21c98b153.png" alt="图10 NWeight on Haswell + SSD" title="图10 NWeight on Haswell + SSD" />

**图10 NWeight on Haswell + SSD**

### 总结 

我们在 Spark 中设计并实现了层次化存储，主要出于两点考虑。首先是技术上的。在数据规模上去以后，I/O 是 Spark 应用的一个常见瓶颈，而 Shuffle Read 又是造成这个问题的首要原因。由于这个阶段的 I/O 随机访问鲜明，单纯的加入机械硬盘往往难以改善。用一块 NVMe SSD 做缓存，很好地解决 Spark 上的 I/O 瓶颈，带来了显著的性能提升。其次，是成本上的考虑。单块 NVMe SSD 外加若干机械磁盘的组合，既满足了绝大数情况下的性能要求，保证了数据存储容量，投入也不至于太大，非常划算。