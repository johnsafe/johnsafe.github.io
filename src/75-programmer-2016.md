## 用虚幻4开发搭积木的 VR 游戏

文/房燕良

虚幻引擎作为国际顶尖水平的3D引擎，一直是很多像我这样的技术人员的向往，感谢Epic Games采取了免费、开源的政策，使得“旧时王谢堂前燕，飞入寻常百姓家”。在当前VR开发如此火热的情况下，虚幻4在VR方面一直保持着技术领先。笔者有幸在虚幻3时代就有机会深入地学习了这款引擎，目前也在从事虚幻4 VR领域的开发工作，所以希望把自己的一点经验分享给更多的对虚幻4引擎技术感兴趣的同学。

<strong>虚幻4 VR开发从何入手</strong>

很多人都听说“虚幻引擎比较难学”，其实我想说“对未知的恐惧是学习新知的最大障碍”，并且虚幻4在易用性上确实已经比前一代有大幅改进。所以，希望对虚幻4引擎技术已经动心的同学，放松心情，勇敢一点，其实没那么难。
无论是VR，还是游戏开发，首先我们都需要对引擎中的概念有一个基础的理解，把基础知识掌握好。这包括基本的编辑器操作、资源管理流程、C++/Blueprint蓝图编程，以及Actor、ActorComponent等基础入门知识。由于大量的入门知识并不适合使用文字表达，所以我正在CSDN学院连载虚幻4入门的系列视频教学课程（http://edu.csdn.net/lecturer/654）。对于没有接触过虚幻4开发的读者，我建议先看看此视频教程，再继续阅读本文。
对引擎有了基本掌握之后，VR开发也就水到渠成了。VR开发在技术上，主要是需要对接主流的VR硬件，包括头戴式显示（HMD）、手柄等。例如Oculus和HTC Vive都提供了自己的SDK，让软件开发者可以访问他们的硬件。而整合这些SDK的工作，虚幻4引擎已经做好了，我们只要开启指定的插件即可（如图1所示）。虚幻4引擎在上层，针对VR硬件提供统一的抽象接口，在下面的示例中我们详细讲解。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a00f3562104.png" alt="图1  虚幻4内置了主流VR硬件支持插件" title="图1  虚幻4内置了主流VR硬件支持插件" />

<strong>在Gear VR上开发搭积木的小游戏</strong>

下面我们通过在Gear VR上运行一个简单的搭积木小游戏，来讲述使用虚幻4开发VR游戏的基本知识。
在这个小游戏中，我们使用视线瞄准一个积木块，然后点击Gear VR头盔右侧的Touch Pad即可拿起积木；这时，转动头盔，拿起的积木块会跟随视线移动；再次点击Touch Pad，将这块积木放下。之后，使用物理刚体模拟，来进行状态更新。你可以尝试把很多积木块堆积起来。

<strong>Gear VR项目创建</strong>

首先我们需要使用C++ Basic Code工程模板来创建一个新的工程，基本设置要选择：Mobile/Tablet，以及Scalable 2D/3D。这里要注意，必须选择C++的工程模板，使用Blueprint模板的话，在打包到Gear VR后将无法正常运行。
然后，必须向引擎中添加一个Oculus签名文件。具体的方法是：
1. 手机使用USB线连接电脑；
2. 使用“adb devices”命令获取Device ID，例如：0915f92985160b05；
3. 打开网址https://developer.oculus.com/osig/，把签名的Device ID粘贴进输入框，然后点Download按钮；
将获取到的文件（例如oculussig_0915f92985160b05）放入：引擎安装目录\引擎版本号\Engine\Build\Android\Java\assets（如图2所示）。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a011406bc6a.png" alt="图2  OSIG文件存储路径" title="图2  OSIG文件存储路径" />
4. 最后，需要在编辑器中打开“Project Settings”->“Android”，修改以下选项：
4-1. Minimum SDK Version：设置为19；
4-2. Target SDK Version：设置为19；
4-3. Configure the AndroidManifest for deployment to GearVR。
这样配置之后，这个UE4项目就可以打包到Gear VR上运行了。

<strong>资源准备</strong>

尽管虚幻4引擎中默认带了一些基本形状的几何体，不过，我还是使用3ds Max生成了一套自己的几何体FBX文件。另外，还有几个表情图标作为这些几何体的贴图，我们将使用不同的表情来标识对象的不同状态。将这些FBX、TGA文件拖入Content Browser中，即可导入。
接下来，我们要在引擎中创建一个材质，如图3所示。请注意，贴图的节点“TextureSample Parameter2D”是一个Parameter，这使得我们可以在运行时改变这个节点的内容。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a01162c47ee.png" alt="图3  积木的材质" title="图3  积木的材质" />

<strong>Gear VR开发基础功能</strong>

首先，创建一个叫作VRPlayerBase的C++类，它从Pawn派生，作为我们的玩家角色。开发Gear VR游戏的话，如果每次测试都要打包到手机上去运行，实在是相当忧伤，因为每次打包的时间……呵呵。所以，我写了一段C++代码，使用鼠标来模拟HMD头盔转动，这样就可以很方便地在编辑器中测试视线焦点相关的操作了。另外，Gear VR Touch Pad相关的操作，也封装了一下，一起放到这个类里面。关键代码如下：
<a target="_blank" href="http://ipad-cms.csdn.net/cms/article/code/3056">代码1</a>
那么，在Gear VR真机运行时，如何使得摄像机跟随头显转动呢？这就简单了，因为引擎已经实现了这个功能，只不过它是实现在PlayerController上，这里设置一下VR Player的转动与Controller一致即可（见图4）。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a01243ada0c.png" alt="图4  Pawn的朝向设置" title="图4  Pawn的朝向设置" />
接下来我们实现视线焦点检测的功能，这部分使用Blueprint来开发，具体情况见图5。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0126e5d6b2.png" alt="图5  视线检测功能" title="图5  视线检测功能" />
在此蓝图中，我们调用了引擎所提供的“LineTraceByChannel”来进行射线检测，见图6。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0129abd6fd.png" alt="图6  LineTraceByChannel" title="图6  LineTraceByChannel" />
我们需要指定这条线段的起点和终点。起点就是玩家的眼睛所在的位置：使用GetActorEyeViewPoint节点取得；终点就是沿着玩家的面朝向（GetForwardVector）一定距离的一个点。LineTraceByChannel有两个返回值，其中“Out Hit”是一个结构体，我们使用Break节点来取出结构体中我们需要的项：射线击中的最近的那个Actor；然后我们检测它是否实现了“BPI_VRObject”蓝图接口（这是我们自己定义的一个蓝图接口，后面详述）；最后我们调用自定义事件：“OnFocusActorChanged”来处理具体的逻辑。
现在，可以设置一个新的GameMode，来指定这个类作为Player Pawn，然后把它设置成默认的GameMode。

<strong>积木块</strong>

创建一个新的蓝图类，用来实现积木块的相关功能，选择从Static Mesh Actor来派生。
首先，为了实现动态改变积木块贴图的功能，要在Construction Script中创建Dynamic Material Instance，如图7所示。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a012d9b4009.png" alt="图7  创建Dynamic Material Instance" title="图7  创建Dynamic Material Instance" />
在图7所示Blueprint脚本中，我使用“CreateDynamicMaterialInstance”节点，为StaticMeshComponent创建了一个动态材质实例，并把它保存到一个名为“BlockMaterial”的变量之中。
另外，考虑到今后可能添加其他的对象类型，创建了一个Blueprint Interface，命名为：用来定义玩家对场景中物体的交互，主要就是图8中的四项。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a013043b73c.png" alt="图8  Blueprint Interface" title="图8  Blueprint Interface" />
接下来，我们做一个简单的功能：当玩家注视着这个积木块时，它就向玩家眨个眼（换张贴图）。首先，要在积木块的Blueprint的Class Settings中，添加上述Blueprint Interface。然后，就可以在Event Graph中使用Add Event菜单，添加OnFocus和LostFocus事件处理了（如图9所示）。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0137d6b88f.png" alt="图9  Focus相关事件处理" title="图9  Focus相关事件处理" />
在上述蓝图中，我们实现了“BPI_VRObject”的两个接口事件，分别是：OnFocus和LostFocus，在焦点获得或者焦点失去的时候，我们调用“SetTextureParameterValue”节点，来改变材质中的BaseColor所使用的贴图资源对象。
接下来，就要实现一个有趣的功能：当积木块坠落的时候，显示一个害怕的表情；当积木块静止不动后，显示微笑表情。这个功能，通过刚体（Rigid Body）的Wake、Sleep状态来实现。从物理引擎的角度说，当物体静止不动时，物理引擎会把这个物体设置到Sleep状态，以节省运算量；当需要物理引擎计算其运动状态时，再把它叫醒。注意：要设置StaticMesh组件的“Generate Wake Event”才能收到这两个事件，见图10。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a013cceb42b.png" alt="图10  刚体的Wake、Sleep事件处理" title="图10  刚体的Wake、Sleep事件处理" />
在这个Blueprint脚本中，我们通过响应“OnComponentWake”和“OnComponentSleep”来变更了自定义变量“IsSleeping”，然后调用自定义事件“ChangeTextureByState”来变更材质的贴图参数。
接下来，还可以做一个小功能：当玩家抓起这个积木时，它就开始哭。这个实现方法和上面的思路完全一样，在此不多赘述。

<strong>在玩家周围随机生成一些积木块</strong>

OK，既然的积木已经准备好了，就可以在关卡启动时，随机地生成一些积木，供玩家玩耍。这个功能可以在Level Blueprint或Game Mode中实现。这里，我们假设这是本测试关卡的功能，所以把它放在Level Blueprint中去实现。
图11展示的这段Blueprint代码，即是在Player Start对象周围随机产生了10个积木块。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0140c9f372.png" alt="图11  在Level Blueprint中随时生成积木块" title="图11  在Level Blueprint中随时生成积木块" />
响应关卡的BeginPlay事件：通过ForLoop节点，调用10次SpawnActor节点，这个节点的Class参数选择成我们的积木块（BP_Block）；积木块的出生点通过MakeTransform节点来生成。在MakeTransform节点中，使用了3种不同的随时方式来产生位置、旋转和缩放这三个参数。

<strong>拿起和放下积木</strong>

接下来，我们继续完善玩家类，添加拿起、放下积木块的功能，具体操作是：玩家首先要注视着某个积木，然后轻点Touch Pad可以拿起积木；转动头盔，积木会随其移动；如果再次轻点Touch Pad，则放下这个积木。
首先，我们在Touch Pad的Tap事件中实现上述流程。注意，这个Tap事件是在VRPlayerBase那个C++类中触发的，见图12。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a01448f1345.png" alt="图12  Touch Pad的Tap事件响应" title="图12  Touch Pad的Tap事件响应" />
在此，我们判断如果PickActor是一个有效值，则调用DropActor，否则的话，调用PickFocusActor。
其中的Pickup Focus Actor和Drop Actor是两个自定义事件。在PickupFocusActor事件中，首先我们通知了这个积木块对象：调用它的OnPickup接口；然后使用一个TimeLine驱动的动画，来控制积木块的位置，使它从当前位置，平滑地移动到玩家面前的一个位置。在DropActor事件中，则首先通知了积木块对象：调用它的OnDrop接口；然后把PickedActor对象设为空值，代码见图13、图14。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0149e8a49b.png" alt="图13  捡起正在注视着的对象" title="图13  捡起正在注视着的对象" />
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a01584d57c3.png" alt="图14 放下正在拿着的积木" title="图14 放下正在拿着的积木" />

<strong>总结</strong>

通过上述过程，一个简单，但还有点意思的积木小游戏就准备好了（见图15）。你可以通过项目打包功能生成APK包，在Gear VR上进行体验。由于篇幅所限，项目中一些细节无法在此完全详述。大家可以从CSDN CODE下载这个项目的完整资源，来进行参考https://code.csdn.net/Neil3D/vrstarter。
本文所演示的项目，只是为了讲述最基本的开发方法，并没有在画面效果上做任何修饰。实际上，虚幻4现在手机上已然可以支持PBR材质效果。总之，各种酷炫的效果，还待大家去发掘。在今后的一段时间内我会持续更新关于虚幻4开发的视频教程和博客，欢迎大家关注http://blog.csdn.net/neil3d。
<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a014ee67ab1.jpg" alt="" title="" />