## 物联网开发中意想不到的那些“坑”



文/刘洪峰

虽然大学开设了物联网专业课，最近也有一批物联网专业的学生毕业了，但是真正学好、做好物联网开发，却不是一件容易的事。从程序开发的角度上来说，既要熟悉嵌入式，也要熟悉桌面或 Web 平台开，同时还要懂手机程序开发。另外，在智能硬件开发比较深入的当下，熟悉智能硬件，能设计智能硬件，连接各种传感器也是必须具备的技能。只有掌握这些技能，才能有比较完整的物联网开发视角，才可能开发出相对实用的物联网系统。

本文先简述笔者的嵌入式开发经历，然后结合最近新开发的一个实验性质的养鸡物联网项目，总结在物联网开发过程中所遇到的那些意想不到的“坑”。

### 从 PLC 开发到鸡舍物联网

如果从上大学开始写 Basic 程序算起，笔者从事软件开发已经20多年了。但是接触所谓的嵌入式硬件是2001年进行 PLC 的开发，当时主要是实现通信功能，没有采用梯形图语言进行开发，而是采用的类似汇编语言的语句表。接着是在2003年开始接触 WinCE 触摸屏开发，采用 C# 和 EVC 进行嵌入式组态开发。后续在2005年左右开始做隧道广告的通信系统，初始采用的是基于 DOS 系统 X86 嵌入板，用 BC3.1 进行开发。另外在焦炉四大机车系统开发中，AB 的 PLC 需要通过一个第三方模块获取机车轨道坐标信息，里面的系统是 TinyDOS，也是采用 BC3.1 进行开发。以上所说，谈不上真正的嵌入式开发，更谈不上硬件开发，最多算是嵌入式应用开发。

2008年在微软.NET Micro Framework 项目组，对 TI DM355 芯片进行.NET Micro Framework 系统进行移植的时候，笔者主要负责 I2C、UART 和 USB 的驱动开发，采用 Insight3 进行代码编写，采用 MDK 和 RVDS 工具进行编译和调试。

在2010年初的时候，利用业余时间率先把.NET Micro Framework 系统移植到 Cortex-M3 架构的芯片上（STM32），并且所有的驱动代码从零写起，全是基于寄存器操作层面进行编写。至此，笔者才觉得真正理解嵌入式系统，才算是迈进嵌入式或智能硬件开发的殿堂。

从那之后，开始设计物联网产品，并且也可以绘制简单的 PCB 板。物联网智能网关、物联网智能终端、物联网智能 I/O 模块和物联网采集模块陆陆续续被设计出来。年前实施的养鸡物联网监控是笔者，软硬件亲自设计、开发，并且到现场安装和调试的首个项目。下面先简单介绍一下该项目。

本系统采用五层架构：传感器/智能设备→采集器/智能终端→智能网关→云中间件/Web 后台→网页/微信。

鸡舍一般需要监控的参数，包括光照、温度、湿度、二氧化碳、氨气、氧气等，此外还要每天监测鸡的重量、水的用量及电的用量等。下面是相关的传感器列表：

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53dd32464e.jpg" alt="表1  传感器列表" title="表1  传感器列表" />

表1  传感器列表

为了便于连接各种传感器，笔者开发设计出了物联网采集模块（如图1），该模块具有1路 RS485 接口、4路模拟量接口、4路串口、4路 I2C 接口和1路 SPI 接口。由于目前 Cortex-M3 芯片支持 GPIO 复用功能，所以一些类似单总线功能都可以支持。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53e518c8a3.jpg" alt="图1  连接各种传感器的物联网采集模块" title="图1  连接各种传感器的物联网采集模块" />

图1  连接各种传感器的物联网采集模块

### 物联网开发的那些“坑”

下面介绍所谓的“坑”——称为“陷阱“似乎更合适，是很难意识到的一种危险，一旦碰上能否顺利脱困真的不好说。比如最开始做硬件的时候，意识不到芯片会有所谓的散新、全新，也没有想到芯片市场竟然也充斥着不少假货。开发以太网功能时，意识不到以太网接口会有十几种之多，网口会有兼容性问题，会和不同的路由器有适配问题。也意识不到 PCB 电路板不仅有两层、四层，更有六层和十层之多，并且不是线连通就没有问题，各种线、层之间的干扰，甚至制板环节出现的问题都会导致最终成品出现各种匪夷所思的问题。

#### 传感器和采集模块层面之间的“坑”
笔者下面所提到的，就是年前实施项目时所遇到的传感器和物联网采集模块层面之间的“坑”。

第一个“坑”，最开始测温度原打算采用 SHT1x 系列的温湿度芯片，后来考察发现，湿度监测点其实只有一个，而温度至少需要四个，所以综合考虑，还是采用了 DS18B20 芯片。由于安装物联网采集模块盒子的防水接头的孔径大概是6mm左右，去现场的时候采购的四芯双绞屏蔽线没有找到这么细的，只好选择了四芯屏蔽线。鸡舍一般120米长，安装在前、中、后三个位置，那么每个温湿度探头平均分布，如果温度采集模块在中间位置，那么至少间隔40米左右相对合适，如果想减少 RS485 的通信距离，物联网采集模块离网关比较近一些，那么最远的通信距离应该在80米左右。为了便于布线，初步打算采用的是总线方式，也就是一根线上同时连接4个传感器（含一个外部环境温度）。但是现场测试发现100米的线根本无法正确获取温度数据，40米左右的线也不行。经过上网查证一些资料，决定把 GPIO 由原来的推挽模式，改为驱动能力更大的开漏模式，同时把上拉电阻尽量变得更小一些。测试发现40米左右可以正确获取温度数据，但是100米仍然不行（查询相关的资料，发现如果是屏蔽双绞线，传输距离会进一步变大，一般会有3倍以上的提升）。考虑到现场条件的制约，最终方案是物联网采集模块放在中间，向两端分别布设约30-40米左右现场的温度探头，以这种方式迈过了第一个坑。

第二个“坑”，同样也是出现在 DS18B20 上。单总线驱动其实是.NET Micro Framework 官方自带的驱动，应用示例来自国外 GHI，以前测试也没有发现问题。现场开始实施的时候也没有问题，后续在观察温度数据中发现，会出现异常大的值。仔细分析后发现，当温度在零下的时候会出现这样的值。查代码发现，原来示例中数值转换的代码有误，看来最初写这个示例的程序员只在温度适宜的地方测试过呀。（附记：后来根据上传到云端的温度曲线，发现温度在-1~1摄氏度的之间，温度会出现跳变，如图2所示。把温度探头放到冰箱，一点点看着温度变到零度以下，并没有发现类似的温度跳变，可惜后续回来后，温度一直没有零度以下，而在冰箱又是快速变到零度以下的，所以很难复现现场的温度趋势，直到现在，还弄不明白为什么会出现这种情况。系统实际运转的时候，鸡舍的温度应该在38摄氏度左右，不会出现零摄氏度以下，但是这是一个值得深究的问题）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53ed229920.jpg" alt="图2  温度跳变曲线" title="图2  温度跳变曲线" />

图2  温度跳变曲线

第三个“坑”，也是温度上出现了问题，这次是采集 SHT1x 芯片温度时出现的，就是低于零摄氏度时，只会显示零。查底层驱动代码发现，当时没有考虑到温度为零下的情况，在对单精度浮点数转双字节的数据变换中，把浮点数的符号位弄丢了。

第四个“坑”，就是称重环节，称鸡的重量，加上围栏和架子，大概200千克左右，要求称重精度几十克。采购了4个 S 型传感器，由于精度要求高，物联网采集模块的芯片 AD 精度最高才12位，远远不够，所以中间添加了 HX711 采集模块，并且在以前测量马路沥青铺设厚度的项目中采用过类似的方案。以为应该没有什么大问题，但是实际测试中发现，除了角差以外，还有如下问题，就是重量和温度相关，并且伴有零点漂移现象（似乎单纯的进行温度补偿不能解决问题），如图3、图4所示。虽然变化的重量分摊到一只鸡上，不过十几克，但是为什么重量逐渐变小，值得深入研究。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53f05ad0fc.jpg" alt="图3  温度变化曲线" title="图3  温度变化曲线" />

图3  温度变化曲线

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53f4b0863f.jpg" alt="图4  重量变化曲线" title="图4  重量变化曲线" />

图4  重量变化曲线

第五个“坑”，由于现场实施时间要求急及水表发货相对较迟，所以到了现场才得以进行水表通信测试。水表默认的通信波特率是1200，而以前测试发现基于.NET Micro Framework 的 Cortex-M3 系统，最低通信波特率是2400（以前监测一款超声波模块就是遇到类似的问题，由于默认的波特率就是1200，所以没有办法通信成功），和水表厂家多次通电话，最终获知，水表是无法调整波特率的，这个固件版本的水表就不含修改波特率的功能。这个时候突然灵光一现，想到移植串口驱动时，COM1、COM6、COM8 的时钟源和其他串口不一样，来源于 APB2 而不是 APB1，APB1 的时钟频率是 APB2 的一半，所以理论上其他串口在分频计算波特率时，可以低一倍。而笔者测试通信一般都是用 COM1 进行测试的，所以决定用 COM2 进行通信测试，真没有想到，竟然通信成功了。换了一个 RS485 接口，解决了水表的通信问题。

除了以上五个“坑”，在传感器/智能设备→采集器/智能终端这个环节，还出现比如选择的互感器线圈，无法正确识别风机的开启和关闭，信号跳变的“坑”。水表的供电范围是9~18V，而现场提供的开关电源只有24V 的“坑”等等。

#### 云端通信的坑
为了简化智能网关和采集器/智能终端的通信，采集器和智能终端对外统一物理接口为 RS485，通信协议为 Modbus。针对每一种传感器模块（内含采集器+传感器），都会独立封装成一个驱动，这样的好处是，相关的传感器数据自动会导入到 YFIOs 嵌入式组态软件中。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5400a37a32.jpg" alt="图5 YFIOs架构" title="图5 YFIOs架构" />

图5 YFIOs 架构

针对当前项目一共需要如下几个驱动：智能终端-8路开关量入模块驱动、智能终端-8路继电器输出模块驱动、水表驱动、电表驱动、综合传感器模块（温湿度、光照、氧气、二氧化碳等）驱动、温度传感器模块（多路温度）驱动、负压采集模块驱动和称重模块驱动等。

物联网智能网关具有一个以太网接口、四个 RS485 接口。以上几种设备分别连接在四个 RS485 接口上。物联网智能网关上运行了基于.NET Micro Framework 系统的 YFIOs 组态运行时。通过配置自动获取各种传感器的数据，然后把采集到的数据自动上传到云端中间件。

YFIOs 是 YFSoft I/O Server 的简称，由三大部分构成，图5所示，一是 YFIOs 运行时，包含内存数据库 YFIODB、内存数据块 YFIOBC、驱动引擎和策略引擎四部分；二是应用模块，包含驱动、策略和 IO 数据三部分；三是 YFIOs IDE 环境（YFIOs Manager），该工具和 Microsoft Visual Studio 开发工具共同完成驱动、策略的开发、配置及部署工作。

这个通信环节出现的“坑”，就出在采集器→智能网关环节上，用 YFIOs Manager 配置各种模块驱动时，发现最后添加了称重模块的驱动后，则整个 YFIOs 运行时会出现异常，部署的模块驱动反射时会出现异常，进一步追踪发现驱动文件的偏移地址变为0，百思不得其解，一度以为这个“坑”迈不过去了。

后来进一步发现，和称重驱动模块无关，最后添加其他驱动也会出现这样的问题。再进一步发现，是因为同一个 RS485 接口连接了多个传感器模块的原因，但是在以前的项目中，一个 RS485 接口曾经连接过若干个物联网智能模块，运行正常，没有发现类似问题。

由于这个功能很重要，无法回避，最后只好采用最费时间的办法，一一在线跟踪 YFIOs Manager 和 YFIOs 运行时代码，才发现问题出在当同一个通道加载不同驱动模块时，驱动程序计数出现了问题，没有累加，当同一个通道连接的都是同类的模块，则不会出现这个问题。但连接不同模块，由于需要多种驱动程序，则会出现这个问题。知道问题点，就好修改了，在 YFIOs Manager 中增加了一行计数累加代码，最终迈过了这个“坑”。

由于在网关→云端中间件这个环节的通信功能，已经在垃圾或污水处理等实际项目中已经运行3、4个月了，是经过实际验证过的，理论上应该不会在出现“坑”了，没有想到这个环节又出现了一个“坑”，一个更意想不到的“坑”。

具体的问题是发现用 YFIOs Manager 监控到的数据和云端 YFCloud 收到数据部分不一致，有错位现象。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d54020efdb5.jpg" alt="图6  YFCloud云端中间件客户端管理界面" title="图6  YFCloud云端中间件客户端管理界面" />

图6  YFCloud 云端中间件客户端管理界面

重新从 YFIOs Manager 导出要上传的 IO 的 XML 文件，在云端后台导入，重复多次，数据依然错误。在以前实施的项目中，从未出现这种问题。

仔细分析也没有看出问题出在什么地方。不过要说区别，区别就是 I/O 变量名称中，出现了 T 和 H 两个一个字母的变量，根据这个信息盘查云端 I/O 操作的 C++ 代码，发现原有嵌入式代码移植到云端平台时，在 I/O 变量名合法性判断时原来的小于号后面多加了一个等于号，所以变量名长度为1的时候，则判断为非法，获取该 I/O 变量类型时，就会出现错误，跳出本次循环，造成统计浮点数变量个数的时候漏记，从而导致数据填充错位。

迈过以上几个“坑”，各种传感器的数据终于可以正常的上传到云端了，并且通过网页和手机微信可以查看各种实时数据、历史数据、历史曲线和监控页面等等。微信界面如图7所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5408255e68.jpg" alt="图7  微信实时数据界面" title="图7  微信实时数据界面" />

图7  微信实时数据界面

经历了四五天“激战”，以为终于可以松一口气了。没有想到微信的实时数据页面又出现了问题，即连续运行5-6个小时之后，微信实时数据页面上的数据将显示为空，更为致命的是，这个时候 Web 后台程序会崩溃。

最初在设计实时数据页面时，打算从历史数据报表中提取。虽然所有项目的 I/O 变量，都可以通过设置 CRON 表达式，来设定每一个 I/O 变量什么时候保存，或定时保存间隔。考虑到项目实际需求及保存的数据量大小，最小的存储间隔设定为5分钟。从用户体验上考虑，这个时间还是太久，盯着手机看，数据5分钟才变化，那感觉也太差了。

从现场网关把数据上传到云端中间件，根据现场实际，可以通过以太网进行数据上传，也可以通过 GPRS、3G 或 4G 进行数据上传，考虑到数据传输的即时性及流量限制，数据上传采用变化才上传的策略（对浮点数据，可以设置变化上传的阀值）。

所以实时数据页面最好可以直接显示上传的数据，这样用户的体验才可能比较好。

这次的“坑”就出现在实时数据页面的服务端程序读取 YFCloud 云端中间件的内存数据库（IODB）时候，会出现异常，并且该异常无法被捕捉，这真是一个棘手的问题。

后来仔细查看了相关的访问代码，感觉应该存在多线程同步访问的问题，添加了相关的同步代码后，就再也没有出现过类似的问题。

### 写在最后

本文盘点了笔者在项目实施过程中所遇到的一些“坑”，希望能给物联网或其他领域的开发者带来一些启发。这里必须要强调的一点是，笔者以上说的一些“坑”，其实都应该在研发阶段就被发现，而不是等到在现场实施时才遇到。并且相关代码如果有变动，至少拷机一周到一个月才能去现场去实施，这样才能保证在条件相对制约的现场，可以快速完成项目的实施和联调。

选取当前这个项目来说明物联网开发过程中所遇到的一些“坑”，一是因为这是一个实验性质的项目；二是实施时间比较紧张，所以相关问题在现场实施过程中集中爆发了，并且遇到的这些“坑”，也很有代表性，值得说一说；三是，这是笔者软硬件设计、编码和现场实施调试，全程参与的项目，可以相对全面地讲述一个典型物联网项目的开发过程中有可能遇到的种种问题点。

一直以来，笔者都持这种观点：物联网是工业以太网的一种外延。物联网很多技术来源于工控，也只有基于工控思维，才能做出可靠实用的物联网项目。从这个角度来讲，本篇文章也许又会让读者产生另外一个误解，以为物联网项目的“坑”主要出现在软件层面。其实做物联网项目，特别是工控项目，最主要的就是稳定，而稳定大部分基于硬件的稳定。要克服硬件层面的“坑“就不是一朝一夕的事了，需要持续地升级迭代，让硬件产品在一个个实际现场项目中得到升华。当然也只有深入熟悉软硬件技术，才能做出最好的物联网产品。

所以，物联网开发永远都在路上… …