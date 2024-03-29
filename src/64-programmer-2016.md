## VR 硬件演进与其游戏开发注意事项



文/王秋林

最近两年虚拟现实（VR）从刚刚走进公众视野到逐渐变得炙手可热，很多不同领域的 IT 开发者都想进入虚拟现实领域。本篇文章将首先讲解 VR 入门所需要学习的知识，然后从 VR 软硬件入手，针对当前的 VR 硬件特征以及 VR 技术本身的特点讲解 VR 游戏开发中的若干注意事项。

### VR 入门基础

最近很多其他行业的 IT 开发人员希望进入虚拟现实领域，想了解如何快速入手。这里简单说明一下：虚拟现实技术属于 3D 显示技术的一个分支，传统的 3D 技术显示在普通的 2D 屏幕上（类似于《魔兽争霸》这类游戏）；而现代的 VR 技术则是指，利用具备了双目视差以及头部运动追踪（头部动作捕捉）技术的立体显示装置（如 Oculus Rift 和 HTC Vive），来对 3D 场景进行展示，这类设备允许用户转动、移动头部，从各种角度对 3D 模型进行观看。

所以对于其他行业的 IT 开发者来说，虚拟现实可以简略地看成是 3D 游戏/应用的一种，进入虚拟现实行业首先需要掌握 3D 技术。虽然 OpenGL 和 DirectX 是 3D 技术的基础，但是直接使用这两种技术进行 3D 开发难度大效率低，并不适合普通开发者入门。这里推荐简单好用的 3D 游戏引擎 Unity3D（一般简称 Unity 或 U3D），其采用 C# 作为脚本语言，简单易学；此外，Unity 本身作为一款 3D 游戏引擎，除了具备对 3D 模型进行渲染、显示的基本功能外，还拥有物理、光照、骨骼动画、音频等 3D 游戏所需的各类系统。

掌握了 Unity 开发技术之后，只需开启 Unity 的“支持虚拟现实”选项，就可以轻松地在 VR 设备上显示 3D 画面了。

当然，这只是 VR 入门的第一步，由于 VR 展示和传统 3D 展示有所不同（比如用户可以自由控制相机，用户看不见鼠标键盘，必须依赖 Xbox 手柄或体感手柄等设备进行输入等），导致 VR 开发和传统 3D 开发存在着很多不同之处。就笔者本身开发过的一款“3D 街区漫游”应用来说，此应用中包含了一个正在建设的街区 3D 模型，目标是让用户在这个 3D 场景中漫游来了解这个街区建成后是什么样子，形成直观的视觉印象。按照传统做法，只要采用第一人称视角，用户操作键盘的方向键（或 WASD 键）控制前进方向，用鼠标控制视角即可（类似《反恐精英》的操作方式）。但是当转为使用 VR 技术后，因为用户看不见键盘而只能使用 Xbox 手柄，通过手柄左摇杆控制角色移动。且由于用户的视角和 VR 头盔直接联动，用户在现实中转头，在虚拟环境中的相机就会同步旋转，所以传统的鼠标失去了作用。当然，控制用户“正前方”的朝向也需要一个输入，而这个输入可以用 Xbox 手柄的右摇杆进行控制。

完成了这些，用户就可以带着头盔拿着手柄在虚拟 3D 街区中漫游了。但是你会发现在通过操作手柄摇杆来移动或转身时会产生一种眩晕感，这是由于人在虚拟环境中的运动和现实中的运动不同步所引起的（后文会详细说明）。于是我们研究了一套改进方案，让用户在虚拟街区中事先设定好的观察点观看，采用手柄按键切换观察点，而瞬间移动的方式很好地避免了眩晕感的产生。

由此可见，即使掌握了 3D 开发技术，也需要在 VR 环境下的人机交互方式中采取合适的方案才能够获得良好的 VR 体验。

### VR 硬件

VR 技术的终极目的在于能够给予用户全身心的沉浸式体验感受。人类可以通过视觉、听觉、触觉、味觉、嗅觉等感觉获取环境信息，这其中，通过视觉获取的信息占据了全部信息量的80%以上。因此目前的 VR 技术主要从视觉入手也就不足为奇了。

而基于双目视差的立体视觉技术更是诞生已超过150年。如图1、图2所示的设备是 David Brewster 于1849年在前人的立体视觉装置的基础上改进而来，在形式上已经十分接近我们目前所使用的 VR 头盔了。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/572819ce0e331.png" alt="图1  David Brewster发明的改良型立体镜" title="图1  David Brewster发明的改良型立体镜" />

图1  David Brewster 发明的改良型立体镜

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/572819ffb5ffc.png" alt="图2  以凸透镜取代立体镜中的镜子" title="图2  以凸透镜取代立体镜中的镜子" />

图2  以凸透镜取代立体镜中的镜子

但是这类设备，只能看到固定的立体图像，图像不会随时间而变化，更不用说这种变化能够根据用户头部的运动做出响应。而作为完整的虚拟现实解决方案，立体画面输出和头部追踪，是必不可少的两项技术。

在人类进入电气化时代后，尤其是二战结束后，伴随着电子计算机的发明，若干基于双目立体视觉的电子设备被发明出来。但是由于需要结合头部追踪以及即时渲染显示输出等技术，在那个电子设备体积大性能低的年代，只能做成如图3所示的这样。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281a392f1cd.png" alt="图3  Ivan Sutherland开发的头戴式显示器" title="图3  Ivan Sutherland开发的头戴式显示器" />

图3  Ivan Sutherland 开发的头戴式显示器

这是由计算机科学家 Ivan Sutherland 在20世纪60年代开发的一款终极显示器，它和天花板相连从而减轻了重量，但是由于其高悬于用户头顶的惊悚造型，该立体视觉系统也被称为“达摩克利斯之剑”。该头戴式显示器是一个真实的虚拟和增强现实设备的最早例子之一。能够显示一个简单的几何图形网格并覆盖在佩戴者周围的环境上。

在后来的发展中，虚拟现实设备一直属于军事研究的重要领域，如用于军事训练，以及顶尖战斗机的头盔里的数据显示。

到了21世纪，若干结合了 3D 显示和头部追踪的设备已经出现了，但是由于其昂贵的价格，往往高校和大企业的实验室才会购置。

#### Oculus 的诞生
革命性的变化出现在2012年，Oculus Rift 登陆 Kickstarter 众筹成功，并于2014年被 Facebook 收购。其实作为第一代产品的 DK1，在现在看来有很多的缺陷，比如分辨率低等，直到2014年发布的 DK2 才有了比较大的改善。

DK2（如图4所示）在2014年北京举办的 Unite 大会上有过展示，参加过这次大会的同学可能还有印象，当时排的队伍之长，至少要等40分钟才能玩上，而看着带着 DK2 玩家的反应非常有趣（当时演示了一个鬼屋游戏，游戏最后会出现一个女鬼，玩家往往会被吓得惊声尖叫）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281a841419b.png" alt="图4  Oculus Rift DK2" title="图4  Oculus Rift DK2" />

图4  Oculus Rift DK2

DK2 具备了相对较高的分辨率（1920x1080，不过即使如此，仍然会看到较为明显的纱窗效应），以及100°左右的视场角（FOV），采用了高精度的陀螺仪结合红外相机进行头部运动追踪，并对从头部运动数据采集到最后的 3D 图像渲染输出的一系列环节进行了优化（甚至据说 DK2 配的 HDMI 线都是特制的，里面有定制化芯片用于缩短视频信号传输延迟）。而其所采用的三星 Note3 手机所配备的 OLED 屏幕具有优良的高刷新率，能够做到 75FPS ，大大减小了液晶显示器显示高速物体时产生的残影现象（该现象是导致 VR 眩晕的原因之一）。可以说，DK2 是当时十分难得的没有致命短板的 VR 设备，且最关键的一点是，它399刀的价格也让人比较容易接受。

#### 惊鸿一瞥的 HTC Vive
而到了2015年，HTC 和 Valve 公司在巴塞罗那的消费电子展上宣布联合推出 Vive 虚拟现实设备，但是由于发出的测试样机过于稀少，VR 游戏开发者普遍还是用 Oculus 进行开发。

#### 中国的 VR
同时在大洋彼岸的中国，VR 也经历了大起大落的一年。一方面，一些公司推出了“蛋壳 VR”，在蛋壳形状的动感座中加入了国产的 VR 头盔，配合“大摆锤”、“过山车”等 VR 游戏，席卷全国一二线城市的大商场。但是由于 VR 游戏对于动感座椅是没有任何控制的，而是由一个外部程序控制座椅的运动，因此游戏和座椅之间难以实现良好的同步，比如当用户在游戏中看到自己开始加速下坠1-2秒之后，座椅才会有相应的动作，这种“不同步”会引发用户产生较为严重的眩晕感。而且不同厂家对驱动 VR 头盔的电脑配置也不尽相同，为了节约成本而配置较差的显卡也会引发低帧率、高延迟，导致用户眩晕。这些都降低了客户的体验感，且少有后续内容更新，也导致了回头客的减少，而蛋蛋椅的销量在2015年10月之后严重下跌，这波“中国式 VR”的浪潮也暂时退去。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281ac954ccc.png" alt="图5  用于体验VR的蛋蛋椅（蛋壳）" title="图5  用于体验VR的蛋蛋椅（蛋壳）" />

图5  用于体验 VR 的蛋蛋椅（蛋壳）

公平来说，虽然“蛋蛋椅 VR”（如图5所示）体验谈不上多好，但是对整体市场起到了一定的教育作用，让大众在一定程度上了解了 VR 到底是个什么技术，但是同时由于不到位的效果，让大众用户感到“VR 也就无非如此罢了”。还有一点就是比起成年人的浅尝辄止，青少年儿童表现出了对 VR 极为浓厚的兴趣。而且对于 VR 眩晕症，儿童表现出了更强的抗性，在我们公司举办的一次 VR 宣讲会上，有一个 VR 模拟飞行的 Demo，一个小孩子玩了半个小时。很多成年人玩过反应会产生眩晕感的游戏，很少听到孩子们玩过后有类似的反馈。

同时借着资本市场对游戏产业（这里主要是指手游）的热捧，手机 VR 伴随着 Google Cardboard 的发布开始了热炒，各种以 Cardboard 为原型设计制造的手机 VR 盒子层出不穷，甚至据说深圳有上百款手机盒子在设计和生产。但是与之形成鲜明对比的却是手机 3D 性能的普遍低下，毕竟由于需要每秒75帧的刷新率，对于一个中小型 3D 场景来说，GTX680 或者 GTX770 以上的台式机显卡性能才能够保持足以稳定的帧率输出，作为手机那捉襟见肘的 3D 性能，如何去实现这样高品质的 VR 体验呢？此外绝大部分手机仍然采用的是不适合 VR 的液晶显示屏，只有三星等少数手机采用了 OLED 屏幕。而且为了满足精确低延迟的头部动作捕捉需要，加速度传感器需要达到 1000HZ 的采样率。但是智能手机中配置的往往是 100HZ 采样率的传感器，结果会大大增加画面相对与头部运动响应的延迟。

此外，这些 VR 盒子往往只是简单容纳手机，缺乏和手机软件进行通信的外部设备。如果用户需要做某些操作，可能需要把手机从盒子里拿出来后点击手机屏幕操作，体验并不好。当然有些软件采用头瞄系统，让用户注视某个虚拟按钮一段时间后相当于按下此按钮。在这一点上，具备了实体操作按键的一体机和 Gear VR 就好很多，操作更加直观便利。

和手机 VR 一起被热炒的，是一系列 VR 一体机的诞生。但是由于这些一体机基本相当于把智能手机去掉了基带芯片整合进手机盒子一样，相比普通手机 VR，2015年的这一批 VR 一体机在性能表现上没有什么竞争力，销售表现也没有掀起太大的波澜。

其实也不是国内一体机厂商不愿做出好产品，有些一体机的创业团队就吐苦水说，作为头盔核心部件之一的 OLED 面板货源只掌握在三星等少数几家大厂手中，无法获得普通液晶面板那样的敞开式供应（估计为了降低风险，一体机厂家的初期产量不会很大，较低的需求量也降低了和三星等供货大厂的议价能力；而作为 VR 产品核心的 OLED 面板，相对于这个产业更是战略级储备资源，在三星手机、Oculus Rift DK2 这些本家产品尚在上升阶段的时候，也很难想像他们会向同行业的竞争对手大批量出售这样的重要战略资源）。而且即使拿到了面板，也不等于万事大吉了，OLED 面板只是负责显示，想让其工作还需要相应的驱动芯片。普通的驱动芯片只能达到 60FPS 的刷新率，只有原厂的驱动芯片能实现和其 OLED 面板所适配的 75FPS 的驱动能力。所以只有原厂的 OLED 搭配自家的驱动芯片才能实现完整的 75FPS 的完整性能表现。至于这些驱动芯片，其获得难度更甚于 OLED 面板。

但是比手机 VR 好的一点大概是，相比智能手机（尤其是 Android 手机）的硬件、系统的碎片化（各种分辨率的屏幕搭配了各种不同规格的处理器，再加上各种版本的 Android 系统，会形成数量恐怖的排列组合结果），一体机有着相对统一的硬件规格和操作系统，VR 软件开发者在进行硬件适配时比起手机 VR 所花费的时间、人力、测试硬件成本会低很多。此外，得益于一体机较大的体型，其中能容纳更多的设备，可添加更多的外部按键、更大容量的电池，甚至是为了防止镜片起雾和给 CPU 更有效的散热而支持搭载基于风扇的主动散热系统。

#### Gear VR
向 Oculus 提供 OLED 面板的三星，和 Oculus 合作推出了 Gear VR（如图6所示）。虽然普通的手机 VR 相比于 PC VR 的表现相差甚远，但是 Gear VR 在手机 VR 之中却是一种异质般的存在——它能够提供基本接近 PC VR 的性能体验。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281b1b96e95.png" alt="图6  Gear VR" title="图6  Gear VR" />

图6  Gear VR

首先，Gear VR 可以通过 USB 接口和三星手机相连（想把别家手机通过这个 USB 接口连到 Gear VR 的同学可以基本放弃了，这个接口被设计成为只能接入三星手机特制的 USB 接口），头盔上的电子设备可以和手机进行直接的有线通信，这些设备包括了光电传感器、触摸面板、高精度陀螺仪等。

当你把（三星）手机放入 Gear VR 后，不用再做任何操作，只要把头盔戴到头上，头盔内置的光电传感器会将信号传递并整合到操作系统层面的 VR 服务，手机就会自动启动 VR 桌面程序。你可以通过触控板浏览当前已有的 VR 应用，并启动选中的应用程序。在游戏中也可以使用触控面板进行操作，如果想要退出该应用只需按下头盔上的“退出”按键即可。

在 VR 桌面，你会发现 Gear VR 对于用户的头部追踪非常准确，头部的运动能够以极低的延迟、极高的帧率直接反应到画面上，而这基本就是 PC VR 的效果。同样是手机 VR，Gear VR 和国产的手机 VR 的差距咋就这么大捏？

首先，在头部运动捕捉环节，刚才已经说过，Gear VR 采用了头盔上搭载的 1000HZ 高精度惯性传感器所采集的数据，并通过有线获得稳定的数据传递输入；国产手机 VR 一般采用手机自带的 100HZ 加速度传感器。这导致了 Gear VR 能够获得更短延迟，更准确的头部运动加速度采样数据。

而在渲染环节，Gear VR 也有独特的优势，三星手机的软硬件都是自己制作的，在 ROM 的开发中，可以对 GPU 的操作做出特殊优化，缩短渲染延迟。而完成这项工作的，则是 3D 技术大神，也是 Oculus CTO John D.  Carmack。正是他参与到三星 ROM 的优化过程中，把 Gear VR 的 SDK 和三星手机的底层显卡驱动进行深度整合优化，实现了 Gear VR 那接近 PC VR 般的梦幻表现。

而国产手机 VR，一般都是使用 Google 的 Cardboard 组件开发的 VR 应用，未经过优化的双镜头渲染让手机 GPU 不堪重负，我在 Unite 2015的展示厅里看到的用于进行虚拟旅游的某手机 VR 应用，其帧率更是只有可怜的个位数，因为我感觉都可以一帧帧自己数出来了（囧）。

Gear VR 到底好到什么程度呢？据说有个国内厂商做一体机研发方案，调研了一大圈，最后决定上三星 Gear VR 的方案，但是 Gear VR 的核心都在三星手机中，想完美复制 Gear VR 就需要自己做一部三星手机出来，但这谈何容易？于是该厂老板做了一个近乎凶残的决定：把三星手机扒皮。也就是买回三星手机，拆去外壳，然后装入自家仿制的 Gear VR 头盔中。其实在一体机硬件方案非常多的“华强北”，搞出这么一个方案只能是无奈的下下之策，但是这里也能看出 Gear VR 其本身的不可替代性。

Gear VR 在手机/一体机 VR 方案里算是鹤立鸡群了，但是它毕竟是手机 VR。当游戏场景中的模型、贴图数量达到一定程度时，也会造成渲染效率的急剧下降，这一点在后面的 VR 开发相关内容中会提及。

此外，Gear VR 还有很大的短板，其应用商店作为一个境外网站因为众所周知的原因在国内无法访问。但是三星目前也在为此努力，希望将来这个局面能够得到改观。

#### HTC Vive 入华
时间到了2015年底，HTC 的 Vive（如图7所示）终于在千呼万唤中登陆中国，在国家会议中心举办了一场发布会，现场排队体验的人更是排成了长龙。和 DK2 采用的“头盔发光－相机采集＋惯性动作捕捉”的头部追踪方案不同，Vive 采用了两个处于对角位置的基站发射不断扫描的红外激光，利用头盔上密布的光学传感器接收激光信号，进而计算出头盔的位置和姿态。该系统同样具备了高精度、低延迟的优秀追踪性能。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281b65cb80b.png" alt="图7  HTC Vive" title="图7  HTC Vive" />

图7  HTC Vive

Vive 具有比 DK2 更高的分辨率（2K 屏幕），更大的追踪范围（Room Scale），以及2个体感手柄。高分辨率的屏幕带来了更小的纱窗效应，而最大5米×5米的可追踪范围让用户可以在一定的空间内移动，以便能够实现玩家在虚拟空间和现实空间中的同步运动，这样彻底消灭了虚拟晕动症的基础，让玩家在使用 Vive 的时候不会眩晕（其实这也取决于游戏内容，后面会提到）。至于体感手柄，采用了和头盔同样的追踪技术，非常精准，甚至有个国外玩家戴着 Vive 头盔，在现实中抛起手柄，由于对于手柄位置极低延迟的精确追踪，虚拟手柄和现实的手柄基本是同步运动的，所以这个玩家仅仅是看着虚拟空间中的手柄，就能够在现实中稳稳的接住这个落下的手柄。

此外由于 Vive 依赖于激光扫描，因此工作环境中应避免有镜子，因为反射的激光会导致系统定位出现问题。

#### 游戏主机 VR
在游戏主机领域，任天堂最早做出的 Virtual Boy 早已折戟沉沙，基于动作捕捉的 Wii，经过长久的时间后回头来看，其平台上的体感游戏除了第一方的游戏以外也乏善可陈；至于推出了 Kinect 的微软，不久前已经传闻解散了多个旗下体感游戏工作室。当然这里并不是说 Kinect 不好，只能说基于“电视＋体感”这样的游戏形式，制作一款真正好玩耐玩的游戏太难了。事实上，基于 Kinect 的低成本 3D 扫描技术在游戏以外的很多领域都得到了应用，其中最直接的就是 Kinect 的技术被整合进了微软的次世代“计算－显示”平台 HoloLens。

索尼在体感游戏领域也是早有探索，在 PS－PS2 时代就有了 PlayStation Eye（简称 PS Eye）；到了 PS3 时代在 PS Eye 基础上诞生了体感手柄系统 PS Move，到了 PS4 时代，面对游戏业未来的 VR 大趋势，Sony 也坐不住了，发布了自己的 VR 企划——Project Morpheus（梦神计划），后来发布的正式名称改为 PlayStation VR，简称 PS VR。其实从技术细节上，PS VR 的头部运动捕捉方案更接近于 Oculus，都是头盔发光，相机拍摄，结合惯性动捕，只不过 Oculus 的 DK2 和 CV1 采用了红外光；而 PS VR 则采用了可见光。这一点也是 PS VR 的一个问题所在，我们日常生活中到处是可见光源，如果在 PS VR 的镜头拍摄范围内有强光源出现，就会干扰捕捉系统，笔者在游艺会体验 PS VR 时就受到了外部光源干扰导致手柄定位失准的情况。

而 PS VR 的分辨率则只达到了 DK2 的程度（1920×1080），这是一个比较聪明的选择。PS4 在2013年发售，其图像处理性能都是在当时的硬件环境下定型的，而无论硬件如何，VR 开发中的 20ms 延迟、75FPS 的标准是一定要坚持的。但是让一个2013年的主机实现这样的性能（事实上 PS VR 宣称自己的刷新率能达到 120HZ），如果采用 2K 甚至更高分辨率的屏幕基本是不太现实的。所以为了最终达成 PS VR 所需的实际性能，据说 Sony 有新款 PS4 的的开发计划，在原有的硬件结构上强化了图像处理能力，但是 Sony 官方目前尚未回应此传言。毕竟一款游戏主机定型后，即使可以针对外形，甚至具体硬件进行改进，但是其最终的硬件性能必须是不变的。这样即使具体硬件型号不同（如 PSP 1000/2000/3000），同一款游戏在这个平台上也会给所有玩家呈以同样的表现。Sony 这样针对原主机推出性能升级型号的行为，很难预测会产生什么样的结果。但 VR 时代对硬件的高性能要求也是不争的事实，老的原则大概也需要考虑新时代的要求。

### 关于 VR 游戏开发

上述当代 VR 设备从本质上来说其实都一样：双目视差 3D 立体视觉＋头部追踪，所以 VR 游戏的开发，很多方面与 VR 硬件来说是互通的。

#### 避免眩晕
VR 眩晕，作为老生长谈的话题，经常被用于评论某 VR 设备的好坏。前面也提到了，动捕系统的延迟、3D 渲染的效率、显示屏的性能都会影响到总的延迟时间和帧速率，而这些指标则直接决定了 VR 体验的好坏。

还有一点，是游戏内容本身。如果在游戏中强行改变玩家在游戏场景中的位置（如过山车、大摆锤或者靠摇杆控制角色移动的第一人称视角游戏），就会引发一部分玩家的强烈眩晕。其原理在于，玩家在视觉中看到了自己在移动，但是身体的各种感觉器官（如耳前庭）却没有感觉到玩家身体的移动信息，这两个矛盾的信息在大脑中引发了眩晕的感觉。而 HTC Vive 就好很多，它的绝大多数 Demo 是让玩家在一定空间内移动，保证玩家的实体头部和虚拟身体头部运动完全一致。

系统性能的低下往往引起延迟、卡顿，给人的体验只是很差，缺乏沉浸感，却不大会引发严重的眩晕；而游戏内容的不恰当设置却会导致玩家出现重度眩晕的感觉。因此，为了避免眩晕，作为开发者，你需要注意游戏的玩法设计；作为玩家，除了使用尽可能好的硬件，游戏内容的选择最为重要，如果你非要使用 Vive 玩过山车，那么无法避免眩晕的结果。

#### 人机交互
在 VR 环境下，原本的 3D 环境＋2D UI 的设计基本行不通了。一则玩家不会再使用鼠标和键盘，可能主要使用 Xbox 手柄或者 Vive 手柄、PS Move 类的体感手柄等；二则 VR 环境下的屏幕分辨率不是线性分布的，越靠近画面中心，分辨率越低，越靠近画面边缘分辨率越高，如果你把 2D UI 放在画面边缘，很可能它会看起来小到玩家根本看不清的尺寸，而且在 VR 头盔里，玩家的眼睛往往只是注视着屏幕中央，很难注视到画面边缘，这也造成了 2D UI 的阅读困难。

而 2D UI 又不能放到画面中间，那样会阻挡玩家在 3D 场景中的视线。所以一般性的解决方案是：采用 3D UI，将 UI 信息融合到 3D 场景之中，使之成为 3D 场景的一部分，而靠体感手柄或 Xbox 手柄的摇杆操作的选择框，可以与 3D UI 对象进行交互操作。

手机 VR 一般缺乏外部输入设备，此时应尽量使用头瞄式输入，避免把按钮设计到屏幕上让用户不得不把手机从盒子中取出后点击该按钮实现输入。

此外，如果能够结合游戏内容，配合现实中的电风扇来实现一些风吹的效果，也会有效地增加沉浸感。

#### 注意游戏优化
由于 VR 游戏对于延迟和帧率的苛刻要求，游戏优化变得非常有必要，尤其是在手机 VR 只能使用性能偏弱硬件的情况下，这种必要性就变得更为突出。比较通用的做法是合并贴图，减小 Shader 种类用于降低 Draw Call；通过遮挡剔除和 LOD 等技术减少需要渲染的三角形数量等；多用低模，在低模上使用法线贴图代替模型表面的实体凸凹来实现低模的精细化效果；通过烘培技术降低实时光的渲染负担等。

#### 注意频闪
在 3D 空间中的等间距的黑白条纹，或者按照一定频率（几十赫兹左右）闪烁的光，可能会诱发某些用户的光敏性癫痫，因此需要注意避免出现这样的条纹或者闪烁。

#### 尽量使用高精细度的贴图
在 VR 游戏中，玩家会比传统 3D 游戏更容易贴近模型表面（甚至玩家可能会很认真地贴近观看模型）。如果模型采用了分辨率较低的贴图，当用户贴近后，瑕疵会一览无余，让用户感觉这个游戏很粗糙。因此结合自身平台的硬件性能特点，在不会显著影响渲染效率的情况下使用分辨率尽可能高的贴图很有必要。

#### 多实际测试
由于 VR 游戏的体验不同于传统 3D 游戏，很多传统的游戏设计方案在 VR 中可能并不可用。因此当游戏策划有了初步的思路后，应该让开发人员尽快制作出简易原型实际体验后验证该思路是否可行。

### 结语

VR 大潮方兴未艾，无数的可能性仍然等待游戏开发者去探寻，希望大家多了解 VR 设备的硬件特性和 VR 游戏开发中的注意事项，充分发挥自己的想像力，制作出好看又好玩的 VR 游戏！

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281c78803b7.jpg" alt="" title="" />