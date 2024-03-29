## FPGA：下一代机器人感知处理器

文/刘少山

十年前，微软创始人比尔 · 盖茨在其文章《A Robot in Every Home》里提出他对未来的憧憬：机器人将会像个人电脑一样进入每个家庭，为人类服务。随着人工智能以及智能硬件在过去几年的飞速发展，到了2016年的今天，笔者坚信各项技术已臻成熟，智能机器人很快进入商业化时代，盖茨的愿景也极有可能在5到10年内实现。

要想机器人有智能，必先赋予其感知能力。感知计算，特别是视觉以及深度学习，通常计算量比较大，对性能要求高。但是机器人受电池容量限制，可分配给计算的能源比较低。除此之外，由于感知算法不断发展，我们还需要不断更新机器人的感知处理器。与其它处理器相比，FPGA 具有低能耗、高性能以及可编程等特性，十分适合感知计算。本文首先解析 FPGA 的特性，然后介绍 FPGA 对感知算法的加速以及节能，最后谈一谈机器人操作系统对 FPGA 的支持。

### FPGA：高性能、低能耗、可编程

与其它计算载体如 CPU 与 GPU 相比，FPGA 具有高性能、低能耗以及可硬件编程的特点。图1介绍了 FPGA 的硬件架构，每个 FPGA 主要由三个部分组成：输入输出逻辑，主要用于 FPGA 与外部其他部件，比如传感器的通信；计算逻辑部件，主要用于建造计算模块；以及可编程连接网络，主要用于连接不同的计算逻辑部件去组成一个计算器。在编程时，我们可以把计算逻辑映射到硬件上，通过调整网络连接把不同的逻辑部件连通在一起去完成一个计算任务。比如要完成一个图像特征提取的任务，我们会连接 FPGA 的输入逻辑与照相机的输出逻辑，让图片可以进入 FPGA。然后，连接 FPGA 的输入逻辑与多个计算逻辑部件，让这些计算逻辑部件并行提取每个图片区域的特征点。最后，我们可以连接计算逻辑部件与 FPGA 的输出逻辑，把特征点汇总后输出。由此可见，FPGA 通常把算法的数据流以及执行指令写死在硬件逻辑中，从而避免了 CPU 的 Instruction Fetch 与 Instruction Decode 工作。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d546968a6c1.jpg" alt="图1  FPGA硬件架构" title="图1  FPGA硬件架构" />

图1  FPGA 硬件架构

#### 高性能

虽然 FPGA 的频率一般比 CPU 低，但是可以用 FPGA 实现并行度很大的硬件计算器。比如一般 CPU 每次只能处理4到8个指令，在 FPGA 上使用数据并行的方法可以每次处理256个或者更多的指令，让 FPGA 可以处理比 CPU 多很多的数据量。另外，如上所述，在 FPGA 中一般不需要 Instruction Fetch 与 Instruction Decode, 减少了这些流水线工序后也节省了不少计算时间。 

为了让读者对 FPGA 加速有更好的了解，我们总结了微软研究院2010年对 BLAS 算法的 FPGA 加速研究。BLAS 是矩阵运算的底层库，被广泛运用到高性能计算、机器学习等领域。在这个研究中，微软的研究人员分析了 CPU、GPU 以及 FPGA 对 BLAS 的加速以及能耗。图2对比了 FPGA 以及 CPU、GPU 执行 GaxPy 算法每次迭代的时间，相对于 CPU，GPU 与 FPGA 都达到了60%的加速。图中显示的是小矩阵运算，随着矩阵的增大，GPU 与 FPGA 相对与 CPU 的加速比会越来越明显。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d546d6af421.jpg" alt="图2  GaxPy 算法性能对比 (单位：微秒)" title="图2  GaxPy 算法性能对比 (单位：微秒)" />

图2  GaxPy 算法性能对比 (单位：微秒)

#### 低能耗

FPGA 相对于 CPU 与 GPU 有明显的能耗优势，主要有两个原因。首先，在 FPGA 中没有 Instruction Fetch 与 Instruction Decode，在 Intel 的 CPU 里面，由于使用的是 CISC 架构，仅仅 Decoder 就占整个芯片能耗的50%；在 GPU 里面，Fetch 与 Decode 也消耗了10%～20%的能源。其次，FPGA 的主频比 CPU 与 GPU 低很多，通常 CPU 与 GPU 都在1 GHz 到3 GHz 之间，而 FPGA 的主频一般在500 MHz 以下。如此大的频率差使得 FPGA 消耗的能源远低于 CPU 与 GPU。

图3对比了 FPGA 以及 CPU、GPU 执行 GaxPy 算法每次迭代的能源消耗。可以发现 CPU 与 GPU 的能耗是相仿的，而 FPGA 的能耗只是 CPU 与 GPU 的8%左右。由此可见，FPGA 计算比 CPU 快60%，而能耗只是 CPU 的1/12，有相当大的优势，特别在能源受限的情况下，使用 FPGA 会使电池寿命延长不少。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5471ec4cb5.jpg" alt="图3  GaxPy 算法能耗对比(单位：毫焦)" title="图3  GaxPy 算法能耗对比(单位：毫焦)" />

图3  GaxPy 算法能耗对比(单位：毫焦)

#### 可硬件编程

由于 FPGA 是可硬件编程的，相对于 ASIC 而言，使用 FPGA 可以对硬件逻辑进行迭代更新。但是 FPGA 也会被诟病，因为把算法写到 FPGA 硬件并不是一个容易的过程，相比在 CPU 与 GPU 上编程技术门槛高许多，开发周期也会长很多。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54759774e3.jpg" alt="图4  传统FPGA开发流程与C-to-FPGA开发流程" title="图4  传统FPGA开发流程与C-to-FPGA开发流程" />

图4  传统 FPGA 开发流程与 C-to-FPGA 开发流程

图4显示了传统 FPGA 开发流程与 C-to-FPGA 开发流程的对比。在传统的 FPGA 开发流程中，我们需要把 C/C++ 写成的算法逐行翻译成基于 Verilog 的硬件语言，然后再编译 Verilog，把逻辑写入硬件。随着近几年 FPGA 技术的发展，从 C 直接编译到 FPGA 的技术已经逐渐成熟，并已在百度广泛被使用。在 C-to-FPGA 开发流程中，我们可以在 C\C++ 的代码中加 Pragma, 指出哪个计算 Kernel 应该被加速，然后 C-to-FPGA 引擎会自动把代码编译成硬件。在我们的经验中，使用传统开发流程，完成一个项目大约需要半年时间，而使用了 C-to-FPGA 开发流程后，一个项目大约两周便可完成，效率提升了10倍以上。

### 感知计算在 FPGA 上的加速

接下来主要介绍机器人感知计算在 FPGA 上的加速，特别是特征提取与位置追踪的计算（可以认为是机器人的眼睛），以及深度学习计算（可以认为是机器人的大脑）。当机器人有了眼睛以及大脑后，就可以在空间中移动并定位自己，在移动过程中识别所见到的物体。

#### 特征提取与位置追踪

特征提取与位置追踪的主要算法包括 SIFT、SURF 和 SLAM。SIFT 是一种检测局部特征的算法，通过求一幅图中的特征点及其有关规模和方向的描述得到特征并进行图像特征点匹配。SIFT 特征匹配算法可以处理两幅图像之间发生平移、旋转、仿射变换情况下的匹配问题，具有很强的匹配能力。SIFT 算法有三大工序：1. 提取关键点；2. 对关键点附加详细的信息（局部特征）也就是所谓的描述器；3. 通过两方特征点（附带上特征向量的关键点）的两两比较找出相互匹配的若干对特征点，也就建立了景物间的对应关系。SURF 算法是对 SIFT 算法的一种改进，主要是通过积分图像 Haar 求导提高 SIFT 算法的执行效率。SLAM即同时定位与地图重建，目的就是在机器人运动的同时建立途经的地图，并同时敲定机器人在地图中的位置。使用该技术后，机器人可以在不借助外部信号（WIFI、Beacon、GPS）的情况下进行定位，在室内定位场景中特别有用。定位的方法主要是利用卡曼滤波器对不同的传感器信息（图片、陀螺仪）进行融合，从而推断机器人当前的位置。

为了让读者了解 FPGA 对特征提取与位置追踪的加速以及节能，下面我们关注加州大学洛杉矶分校的一个关于在 FPGA 上加速特征提取与 SLAM 算法的研究。图5展示了 FPGA 相对 CPU 在执行 SIFT feature-matching、SURF feature-matching 以及 SLAM 算法的加速比。使用 FPGA 后，SIFT 与 SURF 的 feature-matching 分别取得了30倍与9倍的加速，而SLAM的算法也取得了15倍的加速比。假设照片以30FPS的速度进入计算器，那么感知与定位的算法需要在33毫秒内完成对一张图片的处理，也就是说在33毫秒内做完一次特征提取与 SLAM 计算，这对 CPU 会造成很大的压力。用了 FPGA 以后，整个处理流程提速了10倍以上，让高帧率的数据处理变得可能。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54b365dde9.jpg" alt="图5  感知算法性能对比 (单位：加速比)" title="图5  感知算法性能对比 (单位：加速比)" />

图5  感知算法性能对比 (单位：加速比)

图6展示了 FPGA 相对 CPU 在执行 SIFT、SURF 以及 SLAM 算法的节能比。使用 FPGA 后，SIFT 与 SURF 分别取得了1.5倍与1.9倍的节能比，而 SLAM 的算法取得了14倍的节能比。根据我们的经验，如果机器人将手机电池用于一个多核的 Mobile CPU 去跑这一套感知算法，电池将会在40分钟左右耗光。但是如果使用 FPGA 进行计算，手机电池就足以支撑6小时以上，即可以达到10倍左右的总体节能 （因为 SLAM 的计算量比特征提取高很多）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54b640945d.jpg" alt="图6  感知算法能耗对比 (单位：节能比)" title="图6  感知算法能耗对比 (单位：节能比)" />

图6  感知算法能耗对比 (单位：节能比)

根据数据总结一下，如果使用 FPGA 进行视觉感知定位的运算，不仅可以提高感知帧率，让感知更加精准，还可以节能，让计算持续多个小时。当感知算法确定，而且对芯片的需求达到一定的量后，我们还可以把 FPGA 芯片设计成 ASIC，进一步的提高性能以及降低能耗。

#### 深度学习

深度神经网络是一种具备至少一个隐层的神经网络。与浅层神经网络类似，深度神经网络也能够为复杂非线性系统提供建模，但多出的层次为模型提供了更高的抽象层次，因而提高了模型的能力。在过去几年，卷积深度神经网络（CNN）在计算机视觉领域以及自动语音识别领域取得了很大的进步。在视觉方面，Google、Microsoft 与 Facebook 不断在 ImageNet 比赛上刷新识别率纪录。在语音识别方面，百度的 DeepSpeech 2 系统相比之前的系统在词汇识别率上有显著提高，把词汇识别错误率降到了7%左右。

为了让读者了解 FPGA 对深度学习的加速以及节能，我们下面关注北京大学与加州大学的一个关于 FPGA 加速 CNN 算法的合作研究。图7展示了 FPGA 与 CPU 在执行 CNN 时的耗时对比。在运行一次迭代时，使用 CPU 耗时375毫秒，而使用 FPGA 只耗时21毫秒，取得了18倍左右的加速比。假设如果这个 CNN 运算是有实时要求，比如需要跟上相机帧率（33毫秒／帧），那么 CPU 就不可以达到计算要求，但是通过 FPGA 加速后，CNN 计算就可以跟上相机帧率，对每一帧进行分析。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54bd6a3a72.jpg" alt="图7  CNN性能对比 (单位：毫秒)" title="图7  CNN性能对比 (单位：毫秒)" />

图7  CNN 性能对比 (单位：毫秒)

图8展示了 FPGA 与 CPU 在执行 CNN 时的耗能对比。在执行一次 CNN 运算，使用 CPU 耗能36焦，而使用 FPGA 只耗能10焦，取得了3.5倍左右的节能比。与 SLAM 计算相似，通过用 FPGA 加速与节能，让深度学习实时计算更容易在移动端运行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54c047c8c5.jpg" alt="图8  CNN能耗对比 (单位：焦)" title="图8  CNN能耗对比 (单位：焦)" />

图8  CNN 能耗对比 (单位：焦)

### FPGA 与 ROS 机器人操作系统的结合

上文介绍了 FPGA 对感知算法的加速以及节能，可以看出 FPGA 在感知计算上相对 CPU 与 GPU 有巨大优势。本节介绍 FPGA 在当今机器人行业被使用的状况，特别是 FPGA 在 ROS 机器人操作系统中被使用的情况。

机器人操作系统（ROS），是专为机器人软件开发所设计出来的一套操作系统架构。它提供类似于操作系统的服务，包括硬件抽象描述、底层驱动程序管理、共用功能的执行、程序间消息传递、程序发行包管理，它也提供一些工具和库用于获取、建立、编写和执行多机融合的程序。ROS 的首要设计目标是在机器人研发领域提高代码复用率。ROS 是一种分布式处理框架（又名 Nodes）。这使可执行文件能被单独设计，并且在运行时松散耦合。这些过程可以封装到数据包（Packages）和堆栈（Stacks）中，以便于共享和分发。ROS 还支持代码库的联合系统，使得协作亦能被分发。ROS 目前被广泛应用到多种机器人中，逐渐变成机器人的标准操作系统。在2015年的 DARPA Robotics Challenge 比赛中，有过半数的参赛机器人使用了 ROS。

随着FPGA技术的发展，越来越多的机器人使用上了 FPGA，在 ROS 社区中也有越来越多的声音要求 ROS 兼容 FPGA。一个例子是美国 Sandia 国家实验室的机器人手臂 Sandia Hand。如图9所示，Sandia Hand 使用 FPGA 预处理照相机以及机器人手掌返回的信息，然后把预处理的结果传递 ROS 的其它计算 Node。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54c429d233.jpg" alt="图9  ROS在Sandia Hand中对FPGA的支持" title="图9  ROS在Sandia Hand中对FPGA的支持" />

图9  ROS 在 Sandia Hand 中对 FPGA 的支持

为了使 ROS 与 FPGA 之间可以连接，Sandia Hand 使用了 Rosbridge 机制。 Rosbridge 通过 JSON API 来连接 ROS 与非 ROS 的程序。比如一个 ROS 的程序可以通过 JSON API 连接一个非 ROS 的网络前端。在 Sandia Hand 的设计中，一个 ROS Node 通过 JSON API 连接到 FPGA 计算器，FPGA 传递数据以及发起计算指令，然后从 FPGA 取回计算结果。

Rosbridge 为 ROS 与 FPGA 的联通提供了一种沟通机制，但是在这种机制中，ROS Node 并不能运行在 FPGA 上，而且通过 JSON API 的连接机制也带来了一定的性能损耗。为了让 FPGA 与 ROS 更好的耦合，最近日本的研究人员提出了 ROS-Compliant FPGA 的设计，让 ROS Node 可以直接运行在 FPGA 上。如图10所示，在这个设计中，FPGA 了实现一个输入的接口，这个接口可以直接订阅 ROS 的 topic，使数据可以无缝连接流入 FPGA 计算单元中。另外，FPGA 上也实现了一个输出接口， 让 FPGA 上的 ROS Node 可以直接发表数据，让订阅这个 topic 的其他 ROS Node 可以直接使用 FPGA 产出的数据。在这个设计中，开发者只要把自己开发的 FPGA 计算器插入到 ROS-compliant 的 FPGA 框架中，便可以无缝连接其他 ROS Node。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54c8257766.jpg" alt="图10  FPGA成为ROS的一部分" title="图10  FPGA成为ROS的一部分" />

图10  FPGA 成为 ROS 的一部分

最近跟 ROS 的运营机构 Open Source Robotics Foundation 沟通中发现，越来越多的机器人开发者使用FPGA作为传感器的计算单元以及控制器，对 FPGA 融入 ROS 的需求越来越多。相信 ROS 很快将会拿出一个与 FPGA紧 密耦合的解决方案。 

### 展望未来

FPGA 具有低能耗、高性能以及可编程等特性，十分适合感知计算。特别是在能源受限的情况下，FPGA 相对于 CPU 与 GPU 有明显的性能与能耗优势。除此之外，由于感知算法不断发展，我们需要不断更新机器人的感知处理器。相比 ASIC，FPGA 又具有硬件可升级可迭代的优势。由于这些原因，笔者坚信 FPGA 在机器人时代将会是最重要的芯片之一。由于 FPGA 的低能耗特性，FPGA 很适合用于传感器的数据预处理工作。可以预见，FPGA 与传感器的紧密结合将会很快普及。而后随着视觉、语音、深度学习的算法在 FPGA 上的不断优化，FPGA 将逐渐取代 GPU 与 CPU 成为机器人上的主要芯片。