## 基于 Spark 的百度图搜变现系统架构

文/王全，王皓俊，刘少山

最近几年随着智能手机与无线业务的爆发式发展，主流互联网公司都积累了海量来自用户智能手机的多媒体数据，其中蕴含大量有价值的信息。与此同时，最近几年随着机器学习特别是深层神经网络的突破，语音和图像识别技术突飞猛进，让我们终于迈入了一个可以对多媒体文件内部蕴含的信息进行成功理解的时代。如何大规模分布式地执行多媒体分析程序，就成了当务之急。为了服务图搜变现业务，百度在开源项目 Spark 的基础上打造了一套分布式多媒体数据分析系统。传统的分布式计算通常都是以文本格式的日志文件作为输入， 对多媒体数据并无很好的支持。另一方面，业界存在各种图片分析和理解的程序，可谓百花齐放， 但这些程序大都是单机版，没有处理大数据的能力。本文详细介绍了百度分布式多媒体数据分析系统的架构：这个系统前端可以兼容各种多媒体分析程序，做到可插拔。 同时，此系统具备很强的可扩展性，TB 级别的多媒体数据处理基本在一分钟内可以完成。此系统已经在百度大规模上线并稳定运行了一段时间，我们预计不久将开源此系统。

### 图搜变现架构迭代

百度的原有图搜变现 CTR 预估架构是借鉴了百度凤巢广告系统的多年经验搭建起来的。凤巢广告系统主要致力于提升文字广告系统的变现能力，多年来逐步上线了许多专有系统来处理百度广告 CTR 预估流程中的每一步骤。设计思路主要集中于高度优化、高度集成业务逻辑，在性能上达到极致。这套系统搬移到图搜变现产品上，就会产生如下的问题和难点：1. 学习成本和维护成本高，图搜变现的技术人员花费很长时间才把凤巢系统的模块应用到图搜变现的架构中。架构中的每一步骤（ETL、特征提取、模型训练、模型验证等等），都需要专门的集群来支持。2. 研发迭代速度慢，每个新功能的上线都牵涉到多个系统的更新，花费时间长。对于图搜变现这样一个快速发展的产品而言，功能迭代速度希望能够到达以“天”为单位的级别，模型训练更新希望能够达到小时级别，这是现有系统不能满足的。3. 图片特征没有引入到 CTR 预估中。对于图片广告这样一个产品，图片本身的一些特征（比如亮度、颜色等）是影响到用户点击行为的。凤巢系统是基于文字的广告系统，不支持基于多媒体的模型训练。我们在图搜变现上面需要引入图片特征，对于我们的 CTR 预估产生积极的效果。简而言之，我们希望用 Spark 把图搜变现产品从图1左的现有架构升级换代成为图1右展示的新一代系统，代号 SparkONE。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1b1190815.png" alt="图1  图搜变现系统架构迭代" title="图1  图搜变现系统架构迭代" />

图1  图搜变现系统架构迭代

这样的一个新系统带来的明显优势是：1. CTR 预估架构完全基于一个系统，只需要一个集群就能完成所有的流程，研发人员只需要学习一个系统的 API 就能完成各个功能的需求，开发和维护成本大为降低；2. 每个模块之间的联系采用的是统一的 API，新功能开发上许多模块的对接工作大为减少，使得我们的研发重点集中在关键功能的开发上，而不是浪费在系统的对接上面，为快速迭代打下扎实的基础；3. 把图片特征引入模型训练中，提升模型的质量，带动 CTR 预估效果的提升。除此之外，SparkONE 还在积极引入深度学习技术，在我们的模型训练中，希望用户通过 Spark 的通用 API 来调用深度学习库，使得深度学习的使用难度大为降低，同时又提升模型的质量。 

SparkONE 这个系统以百度图搜变现产品为点，以百度多个新产品的智能分析系统为面，最终打造一个简单可依赖的大数据处理系统，满足公司各个新产品的快速迭代需求。从架构上而言，Spark 系统目前只支持基于文本数据的处理，为了引入图片特征的处理，我们需要扩展 Spark 系统对于多媒体数据的处理能力。下面我们就这方面的架构进行详细的说明。

### 处理多媒体文件的动机

传统的基于 MapReduce 的分布式计算通常都以文本格式的日志文件作为输入。在过去的10年里，这种以文本为输入进行分析的方式在很大程度上主导了大数据在众多业务上的应用，比如从日志文件中提取与业务相关的统计量或者用于模型训练，最终以个性化排序或者推荐等方式服务于用户。在百度内部，这种传统的基于日志分析的大数据处理也有着广泛的应用（比如 CTR 预估）并且对公司的核心业务增长有着显著的推动作用，这些都是已经在长期实践中证明过的。

与文本文件相比，解读多媒体文件有很大难度：其一是文本里面已经很清晰明了的很多元信息，本来就是对多媒体文件进行了成功解读之后可能产出的结果的一部分。其二是，如果有了可以对单个的多媒体文件进行解读的程序，面对来自大规模互联网用户的海量的这种多媒体文件，我们目前还缺少一个能够在这样的量级上对于这个解读程序进行高效执行的系统架构。

不难看出上面提到的两点原因并不是彼此独立的，如果在没有一个成熟的多媒体文件解读程序的条件下，就直接谈论如何大规模高效并行执行这样的程序是没有道理的。幸运的是，最近几年随着机器学习特别是深层神经网络的突破，语音和图像识别技术突飞猛进，让我们终于迈入了一个可以对多媒体文件内部蕴含的信息进行成功理解的时代，而在这个新时代里，基于这些理解出来的深层信息的五花八门的真正有实际价值的应用将会是自然而然的。随着第一个技术难关的攻克，如何大规模地高效执行多媒体分析程序，就突然成了当务之急。而这也正式本文接下来会深入讨论的重点。

### 高效处理大规模多媒体文件的必要性和挑战

这里我们以百度的图片搜索业务为例，光是广告业务用到的图片量就有数亿张，而这个基数每天又以百万张的量级在持续增长中。这些海量的图片都需要通过图片的分析和理解程序来取得必要的蕴含在内部的信息。比如要从广告主上传的一系列图片中自动分类出哪些是广告产品的图片，哪些是其他的比如商家用来宣传的二维码。在多张备选的产品图片中，根据从清晰度亮度对比度这样的低级特征到主体识别的高级特征，来找到哪一张才是最适合展现给用户的图片。还有将一些深层次的从图片内容中提取出的特征包含进入到 CTR 预估模型的特征空间中去，这样的特征对于图片搜索中展现广告这样的业务尤其重要，因为用户往往没法观测到图片的其他相关信息而是仅仅根据人眼对图片内容识别的结果来判定是否会去点击广告，如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1b8506e52.png" alt="图2  大规模处理多媒体文件的流程和挑战" title="图2  大规模处理多媒体文件的流程和挑战" />

图2  大规模处理多媒体文件的流程和挑战

在百度图片搜索数亿张图片量级的基数下，很容易看到高效并行处理的必要性。以图片的特征提取为例，对于每张图片的计算时间花费与图片的像素数量和使用的特征提取算法的复杂度有着密切的关系，但一般来说，单张图片的特征提取时间往往是秒级，有些时候对于相对复杂的特征甚至会是分钟级的。这里让我们假设每张图片只需要200毫秒就可以完成特征提取。在一台16核 CPU 计算机上，有一个对特征提取程序的完美多线程实现，如果我们只有一亿张图片需要做特征提取，完成这一亿张图片的特征提取需要花费超过14天的时间。综上所述，面对像百度图片搜索这样量级的多媒体文件输入，在集群上进行大规模的并行处理是必要的，但是随之而来有几个挑战：

1. 首先是给定了单个的多媒体二进制文件和单机单线程的计算条件，如何编写一个分析程序从多媒体文件中分析提取出我们需要的信息。
2. 其次是面对百万、千万，甚至数亿级的多媒体文件，如何高效的执行上面提到的多媒体分析程序。
3. 最后，作为这个功能设计的一个很重要的目标，我们希望能够支持用户用任何语言来编写的多媒体分析处理的核心程序，即插即用，即使这个程序并不是使用分布式计算系统的原生语言来编写的，亦或是这个程序本身从来没有实现甚至考虑过要并行执行。

### 多媒体文件分析和理解的用户核心程序

以图片二进制文件的分析和理解为例，百度的科研团队开发了用于众多不同场景和功能的图片内容分析程序，以下是其中的两个很有代表性的例子：

首先以图片分类的一个应用为例，此客户程序用 C++编写，依赖于百度内部开发的深度神经网络学习 CDNN C++库，而这个 CDNN Lib 又依赖于社区开源的 OpenCV Lib 进行底层的图像读取等操作。输入流是一张一张 JPG 图片，内部首先根据一个预先计算好的图片统计量，对于每一张输入图片的每一个像素计算相对均值的偏差，这个偏差之后就被输入到一个也是事先训练好的多层神经网络中，经过一层又一层的 conv、fc、softmax 等操作，最终给每张图片对于若干个可能的图片分类分别给出置信度分数。程序有两个输出流，一个就是对图片分类的置信度分数，另外一个是根据最终分类结果（最高分数）相应的对原始输入图片进行诸如解析度压缩率等调整，将处理之后的图片输出。顺便说一下这个用户程序就是我们在后文给出评估测试结果时用作 benchmark 的图片分析理解程序。

第二个例子是一个图片特征提取的应用程序，也是用 C++ 编写，有着对底层复杂库的依赖比如 OpenCV。输入仍然是一张一张 JPG 图片，输出只有一个流是多维的图片特征，第一维是类似第一个例子中的分类特征。接下来的维度首先包括一些图片分割后的局部统计特征，包括超像素的最小、最大面积，均值，以及标准差。所有这些抽取出来的图片特征最终被统一为一个流作为程序的输出。

业界的各种图片分析和理解的程序可谓是五花八门，上面举的两个例子也不过是冰山一角。不过这里通过观察共性，我们已经可以得到以下两点很有指导意义的观察结果：首先，图片分析和理解的客户程序通常由 C/C++ 语言编写，底层往往有着相当复杂的对于其他 C/C++ 库的依赖。这些层层的计算机视觉广义上属于人工智能的逻辑，很难全部重写成 Spark 系统原生 Scala、Java 或 Python 语言来直接操作 Spark RDD。所以采取 streaming 的方式，或者用 Spark 的术语来说是 pipe 管道才是比较现实的途径。其次，这些 C++的图片分析程序，在设计和实现的时候通常只有相当有限的基于单机的并行处理能力，很多甚至直接写成是单机单线程的。分布式计算系统应该在不修改这些程序本身的情况下对它们实现并行执行。这些用户程序往往有在运行时对其他库以及预先计算好的结果（比如深层神经网络的模型）的依赖，前者可以通过静态连接来解决，而后者只能是计算系统的责任来分发这些依赖文件。

### 二进制文件流式管道处理

在上一个章节里，我们以两个图片的分析和理解用户程序为例，讨论了这些程序的一些特征、共性和系统需要解决的一些问题。接下我们需要做的就是，给定一个这样的用户程序，如何高效进行执行了。回顾之前在“高效处理大规模多媒体文件的必要性和挑战”中的分析，面对百度图片搜索这样的百万千万甚至数亿级的图片，再理想再优化的多线程单机程序也是没法满足业务需求的，所以把整个的输入图片集合分布到成百上千台机器上进行并行处理是必要的。我们的解决方案总体上看，是在 Spark 上引入了一个新的 RDD 来实现二进制文件流式管道处理。图3就是这个 BinPipedRDD 的总体设计和主要功能。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1bfcb13fe.png" alt="图3  BinPipedRDD的总体设计和主要功能" title="图3  BinPipedRDD的总体设计和主要功能" />

图3  BinPipedRDD 的总体设计和主要功能

我们的输入是一定量的多媒体二进制文件以某种形式存在分布式文件系统上面，而用户想要的输出是所有这些多媒体文件进行分析理解之后得到的信息：比如提取出来的特征或者分类的结果，也可能是希望输出处理变换之后的新的多媒体文件，也可能用户想要几者兼得，计算系统应该都有能力支持。	
如图3所示，我们首先在 Spark Driver 上面将输入的二进制文件集合变换成 String->PDS 的 Pair RDD。这里 Key 是文件名的字符串，Value 的格式是 Portable Data Stream，它并不包含这个文件名对应的二进制文件的全部数据，但需要的时候可以从中将原始数据以 Stream Reader 的形式读出来，在一定程度上可以理解为指向原始二进制文件内容的一个轻量级指针。

之后 Spark Core 会对这个 String->PDS 类型的 RDD 进行输入分割再分配给每个 Spark Executor 去执行。对于每个 executor 来说，拿到的是整个 RDD 的一个分割(Partition/Split)，而且因为输入是基于 stream 的，所以拿到的这个分割也将是很轻量级的，真正重量级的对于内容的读入和处理发生在每个 executor 的 compute 逻辑里面，所以对于不同的输入分割之间将是完全并行没有单点瓶颈的。拿到了输入分割之后，每个 executor 就进入了 BinPipedRDD 的核心 compute 逻辑。在这个 RDD 的 compute 函数里面，总体来说同一个 RDD 里面封装了“两端+三步+四线程”。两端是按照处于用户逻辑执行的之前和之后分为前端和后端，每一端里面分别包含有彼此呼应的三个步骤。四线程分别是输入处理、输出处理、用户逻辑出错处理，最后当然还有用户逻辑本身执行的进程。 

在用户程序线程正式启动之后，就是对于输入处理线程的构造，这一部分因为处在用户程序之前，明显是属于前端的部分，里面包含的三个重要的步骤按照先后顺序分别是对于二进制多媒体文件的 Un-Archive、Encoding 外加 Serialization。

#### Un-Archive
根据 RDD 输入分割里面的每条 record 的 key，也就是文件名的扩展名，判定输入的二进制文件是否被打包，如果曾经被打包，那么进行打包时的 codec。之后调用相应的 codec stream reader 来读取包裹内的每一个二进制多媒体文件的原始内容。注意这一步实质上是对于每一个打包后的二进制包裹文件，恢复成打包前的多个二进制多媒体文件，整个过程都以 stream 的形式发生在内存里面而且是个“一对多”的映射，所以对于不同的 codec 实现上会有相当大的区别。目前我们支持的 codec 包括 zip（同时支持 archive + compression）和 tar (archive only) 。需要注意的是因为大部分多媒体二进制文件都是事先经过强力的甚至是有损压缩的，所以笔者认为我们的 RDD 读入格式能够支持 compression 并没有支持 archive 重要，毕竟在主流分布式存储系统上面存储海量的多媒体文件没有 archive 是不现实的。

#### Encoding
用户的多媒体分析理解程序所需要的输入无非是每一个输入多媒体文件的文件名（字符串）和文件内容（二进制 stream）。为了支持很常见的同一个用户程序线程处理多个输入多媒体二进制文件的情况，我们还需要分别提供文件名和文件内容的长度（都是整数格式）。Encoding 步骤的主要任务就是按照一定的编码规则，将这些格式都统一的转换成 Bytes 的格式，为提交给用户的程序执行来做准备。因为我们需要支持的多媒体分析理解程序通常是用 C/C++编写的，所以我们在系统里默认提供的编码规则也是按照 C/C++的默认规范来的，比如 UTF-8，One byte per char。如果特殊的用户程序有特殊的需要，我们在整个 BinPipedRDD 的最外层还提供了接口，方便用户接入他们想要的定制编码规则，这一点在下面的 API 介绍里还会提到。除此之外，当然还有那些在 encoding 里通常需要注意的问题，比如 big/little endian。一旦发生不兼容的情况，也会造成用户收到的输入完全混乱。好在这部分系统代码与包含在其中的用户程序肯定是在同一台机器上运行的，所以两者都用所谓的 nativeOrder 往往是个不错的选择 。

#### Serialization
上一个步骤的结果是把每一个文件相关的需要提供给用户执行程序的信息都编码成了 Bytes 的格式，接下来这个 Serialization 步骤就是将所有这些 Bytes Arrays 都统一打平成一个 Bytes Stream。这里当然也需要注意序列化中通常需要注意的一些问题，比如写入下一个要出现的 key 的长度的时候，JVM 中默认 Integer 是4 bytes，那么要注意后面做反序列化的时候，也要对应的用4 bytes Int 读出这个长度。 

经过了上面的 Un-Archive、Encoding 外加 Serialization 的三个步骤，输入二进制多媒体文件的 RDD 分割就转换成了一个连续的 Bytes Stream。之后这个 Stream 就直接通过 STDIN 提供给了用户的多媒体文件分析理解程序，这也就完成了前端的所有工作。

顺带提一句，BinPipedRDD 的后端部分基本和前端是逆对应的，它会首先以 Bytes Stream 的形式从用户程序收集 STDOUT，直接产生一个 Bytes 为类型的 Spark RDD。和前端一样，这里涉及到的反序列化和反编码的规则都是支持用户定制的。

图4描述了 BinPipedRDD 的主要功能，接下来我们来看看运行中每一个 Spark Executor 内部的二进制数据流。图中左侧是整个计算的输入和输出，右侧是将每一个输入多媒体二进制文件转换成输出的用户程序。从左上角跟随数据箭头的流动，首先在 Spark 系统里面进行编码和序列化，形成 Binary Byte Stream 交给用户程序，在用户空间里，进行反序列化和反编码，核心的多媒体分析理解程序拿到输入，产生输出流，经过了编码和序列化之后，从用户空间传回给Spark系统，形成 RDD[Bytes]的一个数据分割。图中展示的是对于这个输出数据分割可能的两种处理，collect 回 driver，或者如果输出的是变换后的新多媒体文件，可以选择由系统本身负责进行反序列化和反编码之后进行落盘保存。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1c5fbf16b.png" alt="图4  Spark Executor内部的二进制数据流" title="图4  Spark Executor内部的二进制数据流" />

图4  Spark Executor 内部的二进制数据流

### 设计和实现上的一些考虑

下文将会回顾我们在设计和实现 BinPipedRDD 过程中曾经遇到过的一些问题，引发的思考，以及解决的方式。希望能抛砖引玉，给读者提供一些有价值的参考。部分考虑点在前文已经有或多或少的提及，这里会更清晰地整理出来展开细节。

#### 所有二进制流数据处理并行化
用管道来支持二进制数据流和多媒体文件理解程序的执行，对于分布式计算系统来讲，不可避免大量的计算时间需要花在数据的各种打包编码等相关任务上。从上文提供的信息，我们可以看到 BinPipedRDD 的计算函数里面恰恰包含了所有这些对于输入和输出二进制数据流的 (Un)archive、 (De/En)coding，(De)serialization 的工作。换句话说，BinPipedRDD 所支持的数据输入相当灵活，并没有假设输入的二进制数据流已经按照某种规则做好了编码 and/or 序列化。

这样做的好处有两点。一是在一个专门的 RDD 里面就已经封装好了所有需要做多媒体二进制处理的功能，没有悬挂在外面又要操心强一致强兼容的逻辑，从用户的角度来看会更清晰简洁，更易用。另外一点更重要，既然所有系统和用户的操作都封装到同一个 RDD 里面，显而易见这个 RDD 的计算将会是线性可伸缩的，也就是计算时间会随着输入二进制文件的量线性上升，会随着计算资源 CPU core 的数量线性下降。相对的，如果把比如序列化的操作悬挂独立在外面，就很容遭遇其他麻烦，比如分布式文件系统的海量小文件存储问题，最终成为系统性能的单点瓶颈。

#### 基于流计算的实现使得内存资源占用最小化且不受数据量影响

对于大规模多媒体文件处理，特别是对于底层基于 JVM 的实现，一个很严肃的问题就是大量连续内存的分配可能造成的与 GC 相关的一系列难题。在前面的章节里我们已经提到，从 Spark Driver 分配给 Spark Executor 的输入分割并不是二进制多媒体文件的内容本身，而是一个 Portable Data Stream。值得注意的是，最终这个数据流在经过各种处理和变换，提供给用户的多媒体理解分析程序的时候，也是采取 STDIN 数据流的形式。既然在整个 BinPipedRDD 前端的开始和终止都是数据流，我们在实现的过程中，对于前端内部的三个步骤都采取了流计算的实现方式，每一个步骤内部的操作都是使用一个或者多个 stream reader →stream writer 的方式连接而成。

这样做的好处显而易见，就是输入数据再多再大，在整个系统的 BinPipedRDD 内部也都是一个连续流动的过程而已，没有任何一个步骤会积累数据造成大量内存占用或者更糟糕的——占用的伸缩，实际上每个 stream reader/wrier 占用的只不过的提前分配好的定长 buffer 空间。用一句话概括，就是内存占用将会是恒定且最小化的。这样的基于数据流的实现方式当然也有其弊端，就是如果用户有需要回顾一部分数据，或者比如在模型训练应用中很常见的需要重新过很多次全体数据的时候，这样的实现方式就远没有将所有数据缓存进内存有效了。

#### 灵活的输出格式方便对接各种各样的多媒体理解应用
除了上面提到的灵活输入格式，BinPipedRDD 对于计算结果的输出格式直接采取 RDD[Bytes] 的形式，也提供了更多的可能性，来灵活的适应不同用户程序和应用场景的需求。举几个例子：对于图片挑选的应用，结果 RDD 里存储的很可能是每张图片的分数，最终的操作就是一路 aggregate 这些分数最后把最好的 collect 到 Driver 的过程；对于特征提取或者多媒体文件处理的应用，结果 RDD 里面存储的应该分别是每张图片的特征以及处理后的多媒体文件的二进制内容，最终的操作很可能就是以分布式的方式存储给分布式文件系统。这些典型应用场景都可以很好的支持。

#### 其他考虑：用户定制函数、兼容性、信息交互方式
参考图5中的 BinPipedRDD API 设计，可以看到如果用户需要使用个性化定制的编码/解码/序列化/反序列化的函数，无论是出于性能或者兼容性考虑，都可以很方便地采取函数指针的方式提供给 BinPipedRDD。而系统在执行 RDD compute 的时候就会用这些用户定制的函数来替代默认的逻辑。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1cdac9294.png" alt="图5  BinPipedRDD API" title="图5  BinPipedRDD API" />

图5  BinPipedRDD API

此外，我们在 BinPipedRDD 功能对接产品的过程中，发现某些考虑到分布式执行的用户多媒体分析理解程序，会有获取分布式系统的一些内部信息的需要，比如当前所在的 Partitoin、Task、Attempt 的编号，用来指导计算或者输出。这些用户需求抛开在分布式计算的大环境下合理性正确性的复杂讨论不提，我们还是出于兼容性的考虑给予了支持。不过这样又带来了新的问题，就是这些额外的信息应该以什么样的方式进行交互。目前支持的两种信息交互方式是：对于现存的早就写好难以更改的应用，我们会把这些系统内部信息输出给环境变量，然后用户程序，更可能是封装脚本，读取环境变量来获取。对于新兴的正在开发中的应用，我们鼓励不引入这样的外部依赖，统一走管道。也就是定制一个叫 printPipeContext 的函数（见图5），把这些额外信息作为 stream header 发送给用户程序。

### 性能评估

在真实的生产环境下，通常是大规模的公用集群搭建在资源管理系统之上，处理大规模的输入数据，这个时候各种资源的不足和不断的动态调整是家常便饭，而且集群和数据规模大了之后慢节点问题，底层存储问题和失败任务进行重试的等情况也是不可避免的。我们将百万数量级的图片作为输入，在一个基于 Yarn 管理处于复杂生产环境下的 Spark 集群上，请求10000 CPU 核作为计算资源，运行同样的图片分析理解程序对百万级图片进行处理，经过反复实验发现总计算时间是可扩展的，在使用2000核的时候，耗时大约130秒，在用10000核的时候，耗时30秒左右。

让我们来假设每张图片只有一百万个像素（就是大概1000*1000的图片尺寸，按照今天手机拍照的标准也已经是很小了），每个像素 是 RGB 三个通道，色彩空间是32 bit。那么抛开 JPG 的 header 和其他 codec 相关的 overhead 不谈，总数据量已经约是12 TB 了。当然处理多媒体的数据总量和处理文本日志的情况下并没有直接的可比性，不过毕竟我们的确是在对于展开后达到这样量级的数据，以秒级的计算效率，在进行 pixel by pixel 最终 byte by byte 的理解和处理。

### 结论和展望
本文我们介绍了基于 Spark 的分布式多媒体数据分析系统，填补了大规模高效分析理解多媒体数据的空白。此系统前端可以兼容各种多媒体分析程序，做到可插拔，同时具备很强的可扩展性。目前此系统很好地支持着百度的图搜变现业务，每天处理 TB 级的数据。从功能与性能角度，本系统还有很多扩展与优化的余地。比如初步分析后的数据可以对接深度学习模型来提取更丰富的特征。从性能来说，多媒体数据涉及高并行的数据处理，我们可以引入 GPU 以及 FPGA 加速。在本文完成时，这些工作正在进行中，并已经取得了很好的初步效果，期待在不久的将来再跟大家分享下一步的成果。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1d27592b5.jpg" alt="" title="" />