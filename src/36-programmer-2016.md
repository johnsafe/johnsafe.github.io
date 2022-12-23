## HDFS EC：将纠删码技术融入 HDFS

文/李波

在 HDFS 中，可靠性通过多副本的方式来实现，从而较低的存储利用率成为时下基于 HDFS 应用的主要问题之一。本文将详细介绍 HDFS 的一个新特性——Erasure Coding（EC）， 它在保证同等（或更高）可靠性的情况下将存储利用率提高了近一倍。

### 背景

近些年，随着大数据技术的发展，HDFS 作为 Hadoop 的核心模块之一得到了广泛的应用。然而，随着需要存储的数据被越来越快产生，越来越高的 HDFS 存储利用率要求被提出。同时对于一个分布式文件系统来说，可靠性又必不可少。因此，在 HDFS 中每一份数据都有两个副本，这也使得存储利用率仅为1/3，每 TB 数据都需要占用 3TB 的存储空间。因此，在保证可靠性的前提下如何提高存储利用率已成为当前 HDFS 应用的主要问题之一。

纠删码技术起源于通信传输领域，后被逐渐运用到存储系统。它对数据进行分块，然后计算出一些冗余的校验块。当一部分数据块丢失时，可以通过剩余的数据块和校验块计算出丢失的数据块。Facebook 的开源项目 HDFS-RAID 在 HDFS 之上使用了纠删码技术。HDFS-RAID 对属于同一文件的块分组并依次生成校验块，将这些校验块形成独立的文件，并与原始的数据文件一一对应。RaidNode 作为一个新的角色被引入进来，它负责从 DataNode 中读取文件的数据块，计算出校验块，并写入校验文件中；同时，它还周期性地检查被编码了的文件是否存在块丢失，如有丢失则重新进行计算以恢复丢失的块。HDFS-RAID 的优点是其构建于 HDFS 之上，不需要修改 HDFS 本已复杂的内部逻辑，但缺点也显而易见：校验文件对用户是可见的，存在被误删除的可能；依赖于 MySQL 和 MapReduce 来存储元数据和生成校验文件；RaidNode 需要周期性地查找丢失的块，加重了 NameNode 的负担；使用的编解码器性能较差，在实际应用中往往不能满足要求。另外，由于缺乏维护，HDFS 已将 HDFS-RAID 的代码从 contrib 包中移除，这给使用 HDFS-RAID 带来不少困难。

2014下半年，英特尔和 Cloudera 共同提出了将纠删码融入到 HDFS 内部的想法和设计（HDFS EC），随后吸引了包括 Hortonworks、华为、Yahoo!等众多公司的参与，使之成为 Hadoop 开源社区较为活跃的一个项目。将纠删码融入到 HDFS 内部带来了诸多好处：不再需要任何的外部依赖，用户使用起来更为方便；其代码成为 HDFS 的一部分，便于维护；可以充分利用 HDFS 的内部机制使性能得到最大程度的优化。纠删码的编解码性能对其在 HDFS 中的应用起着至关重要的作用，如果不利用硬件方面的优化就很难得到理想的性能。英特尔的智能存储加速库（ISA-L）提供了对纠删码编解码的优化，极大地提升了其性能，这一点将在实验部分做详细的阐述。HDFS EC 项目期望实现的功能包括：

1. 用户可以读和写一个条形布局（Striping Layout，定义见下文）文件；如果该文件的一个块丢失，后台能够检查出并恢复；如果在读的过程中发现数据丢失，能够立即解码出丢失的数据，从而不影响读操作。
2. 支持将一个多备份模式（HDFS 原有模式）的文件转换成连续布局（Contiguous Layout，定义见下文），以及从连续布局转换成多备份模式。
3. 编解码器将作为插件，用户可指定文件所使用的编解码器。

由于 HDFS 的内部逻辑已经相当复杂，HDFS EC 项目将分阶段进行：第一阶段（HDFS-7285）已经实现第1个功能，第二阶段（HDFS-8030）正在进行中，将实现第2和第3个功能。本文将主要阐述第一阶段（HDFS-7285）的设计与实现。

### 相关概念

#### 纠删码（Erasure Code）与 Reed Solomon码

在存储系统中，纠删码技术主要是通过利用纠删码算法将原始的数据进行编码得到校验，并将数据和校验一并存储起来，以达到容错的目的。其基本思想是将 ｋ 块原始的数据元素通过一定的编码计算，得到 ｍ 块校验元素。对于这 ｋ+ｍ 块元素，当其中任意的 ｍ 块元素出错（包括数据和校验出错），均可以通过对应的重构算法恢复出原来的 ｋ 块数据。生成校验的过程被成为编码（encoding），恢复丢失数据块的过程被称为解码（decoding）。

#### Reed-Solomon（RS）码

存储系统较为常用的一种纠删码，它有两个参数 k 和 m，记为 RS(k,m)。如图1所示，k 个数据块组成一个向量被乘上一个生成矩阵（Generator Matrix）GT 从而得到一个码字（codeword）向量，该向量由 k 个数据块和 m 个校验块构成。如果一个数据块丢失，可以用(GT)-1乘以码字向量来恢复出丢失的数据块。RS(k,m)最多可容忍 m 个块（包括数据块和校验块）丢失。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9ca3dcafee.png" alt="图1  RS(4,2)编码过程示意图" title="图1  RS(4,2)编码过程示意图" />

图1  RS(4,2)编码过程示意图

#### 块组（BlockGroup）

对 HDFS 的一个普通文件来说，构成它的基本单位是块。对于 EC 模式下的文件，构成它的基本单位为块组。块组由一定数目的数据块加上生成的校验块构成。以 RS(6,3)为例，每一个块组包含1-6个数据块，以及3个校验块。进行 EC 编码的前提是每个块的长度一致。如果不一致，则应填充0。图2给出三种不同类型的块组及其编码。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9ca84cd535.jpg" alt="图2  三种类型块组及其编码" title="图2  三种类型块组及其编码" />

图2  三种类型块组及其编码

#### 连续布局（Contiguous  Layout） VS 条形布局（Striping Layout）

数据被依次写入一个块中，一个块写满之后再写入下一个块，数据的这种分布方式被称为连续布局。在一些分布式文件系统如 QFS 和 Ceph 中，被广泛使用的是另外一种布局：条形布局。条（stripe）是由若干个相同大小单元（cell）构成的序列。在条形布局下，数据被依次写入条的各个单元中，当一个条被写满之后就写入下一个条，一个条的不同单元位于不同的数据块中，如图3所示。 

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9caca0a6c4.jpg" alt="  图3  连续布局与条形布局示意图" title="  图3  连续布局与条形布局示意图" />

图3  连续布局与条形布局示意图

### 总体设计

#### 布局的选择

对 HDFS EC 来说，首要的问题是选择什么样的布局方式。连续布局实现起来较为容易，但它只适合较大的文件。另外，如果让 client 端直接写一个连续布局文件需要缓存足够的数据块，然后生成校验块并写入，以 RS（6,3），blockSize=128M 为例，client 端需要缓存 1.12G 的数据，这点决定了连续布局的文件更适合由普通文件转化而来，而条形布局就不存在上述缺点。由于一个条的单元往往较小（通常为 64K 或 1M），因此无论文件大小，条形布局都可以为文件节省出空间。client 端在写完一个条的数据单元后就可以计算出校验单元并写出，因此 client 端需要缓存的数据很少。 条形布局的一个缺点是会影响一些位置敏感任务的性能，因为原先在一个节点上的块被分散到了多个不同的节点上。

HDFS 最初就是为较大文件设计的分布式文件系统，但随着越来越多的应用将数据存储于 HDFS 上，HDFS 的小（即小于1个块组）文件数目越来越多，而且它们所占空间的比率也越来越高。以 Cloudera 一些较大客户的集群为例，小文件占整个空间的比例在36%-97%之间。

基于以上分析，HDFS EC 优先考虑对条形布局的支持。下面的设计与实现也主要围绕已经实现了的条形布局展开。

#### NameNode 端扩展

在 EC 模式下，构成文件的基本单位为块组，因此首先需要考虑的是如何在 NameNode 里保存每个文件的块组信息。一种比较直接的方法是给每个块组分配一个块 ID，同时用一个 Map 来记录这个 ID 与块组信息的映射，每个块组信息包含了每个内部块的信息。对小文件来说，这种做法将增加其在 NameNode 中的内存消耗。以 RS（6,3）为例，如果一个文件比6个块略小些，那么 NameNode 必须为它维护10个 ID（1个块组 ID、6个数据块 ID 和3个校验块 ID）。在小文件数目占优的情况下，NameNode 的内存使用将面临考验。

一个块 ID 有64位，这里将第1个位作为 flag 来区分块的类型：如果为1，则为 EC 块（条形布局的 EC 块，连续布局将在第二阶段考虑）；如果为0，则为普通块。对 EC 块来说，会将剩下的63位分成两部分：最后的4位用来标识内部块在块组中的位置，前面的59位用来区分不同的块组。块组 ID 等同于第0个内部块 ID，其他的内部块 ID 可由块组 ID 加上其在块组中的位置索引得到，比如第0个内部块 ID 为0xB23400（也即块组 ID），那么第3个内部块的 ID 为0xB23403。由于只是用最后4位来区分一个块组中的内部块，因此对一个块组来说，系统目前支持最多16个内部块。

这样一来就可以尽可能地利用 HDFS 当前的机制来实现对块组的支持。块组依旧用类 Block 来表示，其中的三个成员：blockId 代表块组 ID；numBytes 代表块组大小，即所有数据块大小之和，不包括校验块的大小；generationStamp 代表块组的生成时间戳，所有内部块共享块组的时间戳。这里可以根据块组的 Block 对象得到内部块的 Block 对象：blockId 由块组 ID 加上该内部块在块组中的位置索引；numBytes 可由块组大小和该内部块位置索引计算出来；生成时间戳等同于块组的时间戳。每个内部块所在的 DataNode 信息会存储在其所属块组在 BlocksMap 中对应的 BlockInfo 对象中。当一个块被 DataNode 报告给 NameNode 时，NameNode 可以通过其块 ID 判断该块的类型，如果为 EC 块则将最后4位清零便得到块组 ID。通过这种方式，我们只存储了一个块组 ID，大量内部块的 ID 通过计算得到。内部块的大小和时间戳也可由块组信息得到，从而大大减少了 NameNode 内存的占用，如图4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9cb3a500f4.jpg" alt=" 图4  NameNode端扩展" title=" 图4  NameNode端扩展" />

图4  NameNode 端扩展

当 client 写满一个块组或刚开始写数据时，它向 NameNode 申请一个新的块组。新的块组保存于 INodeFile 中并返回给 client。新块组包含各个内部块的 DataNode 信息，它们由 BlockPlacementPolicy 确定。这里需要各个内部块尽可能的分布在不同的 DataNode 或机架上，以免造成多个内部块同时丢失而加大数据丢失的风险。因此，HDFS EC 需要使用符合自己要求的 BlockPlacementPolicy。

NameNode 中有一个守护线程 ReplicationMonitor，它会周期性地执行数据块的备份和删除任务，这些任务由类 UnderRepliationBlocks 和 InvalidateBlocks 来维护。这种方式也非常适合 EC 任务，包括丢失块的恢复，块在多备份模式和 EC 模式之间的转换等。

UnderReplicatedBlocks 类负责对副本数不足的块进行复制。它包含多个优先级队列，用于区分不同复制任务的紧急程度。我们可以将一个块组的副本数定义为其数据块和校验块数目之和，如果其中的一个内部块丢失，就将其副本数减1，这样其副本数就不足，就可以放入到 UnderReplicatedBlocks 的某一个队列中。可以根据丢失的内部块的数目来决定加入到哪个优先级队列中。当选定的 DataNode 传来心跳时，NameNode 向该 DataNode 发送一个 BlockECRecoveryCommand，DataNode 接收到该命令将启动一个恢复任务。需要注意的是，当发现一个块组缺损后未必立即启动恢复任务，因为恢复任务会消耗大量的网络带宽，以 RS（6,3）为例，承担任务的 DataNode 需要读取6个内部块用于解码工作。如果一个集群每天有1%-2%的节点宕掉，立即启动恢复任务可能会耗尽系统的带宽。因此，恢复任务的执行需要配合一定的策略，例如，优先执行缺损厉害的块组，每天限定恢复任务的数量，在系统空闲时启动恢复任务等。

#### Client 端扩展

用户在写一个文件前可以先指定该文件为 EC 模式，也可以对一个文件夹指定 EC 模式，然后所有写入该文件夹的文件都默认为 EC 模式。

**client 写**

HDFS client 通过输出流 DFSOutputStream 向文件系统写入数据。DFSOutputStream 的实现较为复杂，为了能够对其进行功能扩展，我们对该类进行重构，将内部类 DataStreamer 和 Packet（重命名为 DFSPacket）独立出来，使得各个类都有清晰独立的功能，从而为实现 EC 模式下的写操作提供便利。

当写一个条形布局的文件时，需要将数据分散地写到多个 DataNode 上，为此，我们实现了 DFSOutputStream 的子类 DFSStripedOutputStream，它拥有多个并发的 DataStreamer。图5给出了 DFSStripedOutputStream 的内部工作原理。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9cb9cccf11.jpg" alt="图5  DFSStripedOutputStream工作原理" title="图5  DFSStripedOutputStream工作原理" />

图5  DFSStripedOutputStream 工作原理

数据以条形布局的方式写入到 CellBuffers 中。CellBuffers 拥有多个缓存，每个缓存对应条中的一个单元，也对应着一个 DataStreamer。当一个条中的数据单元写完之后，DFSStripedOutputStream 会立刻计算出校验单元并写入到校验块缓存中。当一个缓存中的数据能够装满一个 DFSPacket 时，该缓存就将数据装入一个 DFSPacket 并传给该缓存对应的 DataStreamer。当数据写完准备关闭文件时，最后的一个条可能数据单元没有写满，这时需要对数据单元补零后生成该条的校验单元并写入校验块缓存中， 然后生成最后的 DFSPacket 并将它们发送至各自的 DataStreamer。

在写一个新的块组前，DFSStripedOutputStream 会向 NameNode 申请分配一个新块组。NameNode 会返回一个 LocatedBlock 对象，该对象包含每个内部块的 DataNode 信息。DFSStripedOutputStream 会解析该对象，生成出各个内部块然后发送给 Coordinator。Coordinator 负责各个 DataStreamer 与 DFSStripedOutputStream 的协调工作：DataStreamer 会等待 Coordinator 传递给它一个内部块用于创建数据流通道；当 DataStreamer 开始工作后，DFSStripedOutputStream 便会在 Coordinator 上等待各个 DataStreamer 的工作结果；DataStreamer 写完一个内部块后会向 Coordinator 发送一个结束块，以汇报此次工作的结果（如写入多少字节的数据）；Coordinator 搜集到所有的结束块后汇报给等待在其上的 DFSStripedOutputStream；DFSStripedOutputStream 据此来决定后续步骤，如果太多的 DataStreamer 失败，则结束写操作并返回，否则转向处理 CellBuffers 中新的数据。

DFSStripedOutputStream 在写过程中可以容忍一定数目的 DataStreamer 失败。以 RS（6,3）为例，如果在写一个块组时，失败的 DataStreamer 不超过3个，那么失败的内部块在以后读取时被计算。 当下一个块组到来，失败的 DataStreamer 又可以重新开始写一个新的块。

**client 端读**

client 读一个条形布局文件的逻辑要相对简单。由于数据源分布在多个 DataNode 上，因此在进行读的时候需要连接多台数据块所在的 DataNode。 读操作的扩展功能由 DFSStripedInputStream 实现，它继承了 DFSInputStream。DFSStripedInputStream 以条为单位进行读取。

client 在写的时候，如果一个块组中只有较少的内部块写操作失败，client 会继续写下去。因此，client 在读的时候就会遇到个别数据块丢失的情况。为了能使读操作进行下去，client 需要连接一定数目的校验块，读取相应的校验数据并通过解码得到丢失的数据。这种情况下，client 需要连接更多的 DataNode 以获取参与解码的校验数据，并且解码也会消耗 client 一定的 CPU 资源。为了减少这种情况的发生，我们需要在后台检测有缺失的块组并进行恢复。

#### Datanode 端扩展

DataNode 端的扩展主要是为了实现后台对丢失数据块的解码，以及对数据块进行编码生成校验块，编码部分将在第二阶段实现。

图6展示了 DataNode 端的扩展。为了独立地处理 EC 相关任务，我们在 DataNode 中添加了一个新的类 ErasureCodingWorker。该类维护了一个线程池，每当有一个解码任务到来时，便会将任务交由 ReconstructAndTransferBlock 线程处理。ReconstructAndTransferBlock 线程从若干个 DataNode 读取解码所需要的数据，执行解码计算，然后将恢复出来的块保存到目标节点上。一旦任务完成，ErasureCodingWorker 会向 NameNode 发送一个确认。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9cbf688265.jpg" alt="图6  DataNode端扩展" title="图6  DataNode端扩展" />

图6  DataNode 端扩展

### 性能测试

纠删码的编解码是非常消耗 CPU 的，如果不对其进行优化则很难满足实际应用的要求。英特尔的开源智能存储加速库 ISA-L 通过利用硬件的高级指令集（如 SSE、AVX、AVX2）来实现了编解码的优化。ISA-L 同时支持 Linux 和 Windows 平台。

在 HDFS EC 中，我们实现了两种形式的 Reed-Solomon 算法：一种是纯 Java 实现，另一种是基于英特尔的 ISA-L。我们将在实验中比较这两种实现的性能，同时参与比较的还有 HDFS-RAID 中的实现。所有的实验都选择使用 RS（6,3），这也是 HDFS EC 中的默认值。

图7显示了内存中各个编解码器的性能比较。从图中我们可以看出，ISA-L 的性能是 Java 的4-5倍，是 HDFS-RAID 的20倍左右。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9cc459b659.jpg" alt="图7  不同编解码器性能比较" title="图7  不同编解码器性能比较" />

图7  不同编解码器性能比较

图8给出了不同编解码器 HDFS I/O 的性能对比。该实验运行于一个11节点的集群上（1个 NameNode，9个 DataNode，1个 client），节点间的网络带宽为 10GigE。实验方法为在 client 节点上向 HDFS 写和读一个 12GB 的文件。为了测试解码性能，我们在读之前先杀死两台 DataNode。从图8可以看出，基于 ISA-L 的编解码器的性能均远远优于其他编解码器，相对于 New Java Coder 来说，ISA-L 写速度是其6倍，读速度是其3.5倍。条形布局文件的一个块是分布在不同的 DataNode上，其读和写都可以并发地进行，从理论上讲，其读写性能应该高于3备份模式。但从实验数据上看，仅 ISA-L 的性能远远高于3备份模式，其他的编解码器都低于3备份模式，由此我们可以看出，编解码运算有可能成为 HDFS 读写条形布局文件的一个瓶颈，而 ISA-L 对编码器的优化则消除了这个瓶颈。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9d5ce94f83.jpg" alt="图8  HDFS I/O性能比较" title="图8  HDFS I/O性能比较" />

图8  HDFS I/O 性能比较

从上述的实验数据和分析可以看出，如果集群是架构在英特尔平台的服务器上，那么使用了 ISA-L 的编解码器是最好的选择。

### 项目进度与计划

HDFS EC 第一阶段实现了对条形布局的支持。用户可以读和写一个条形布局的文件，如果发现一个内部块丢失，后台会进行恢复工作。第一阶段代码已经进入 trunk，并计划在2.9或3.0版本中发布。第二阶段我们将实现对连续布局的支持。当前我们的编解码器默认使用的 Reed-Solomon，将来会添加更多的编解码器如 HitchHiker、LRC 等。用户也将可以灵活配置文件所对应的编解码器。

### 总结

将纠删码技术融入到 HDFS 中，可以保证在同等（或者更高）可靠性的前提下，将存储利用率提高了一倍，这将大大减少用户硬件方面的开销。编解码运算要消耗大量的 CPU 资源，而基于英特尔 ISA-L 库的编解码器极大地提高了编解码性能，从而使得编解码的计算不再成为瓶颈。