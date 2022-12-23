## 使用 Unity 开发 HoloLens 应用



文/张昌伟

>开发者们在陆续收到 HoloLens 开发者版的同时，也都着手了 HoloLens 应用的开发工作。本文作者从空间映射、场景匹配、自然交互等核心特性开始，以实践详解了如何使用 Unity 引擎开发一个简单的 HoloLens 应用，并对自己的开发经验进行总结和分享。

### HoloLens 概述
经历数个月的期待与等待，笔者终于拿到了预订的 HoloLens 开发者版本套件。作为市面上第一款发售的 AR/MR 设备，HoloLens 开发者版本具有很多独特的黑科技。今天，我们就来了解 HoloLens 的开发特性（参见图1应用场景）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d600a69733.png" alt="图1 Hololens应用场景" title="图1 Hololens应用场景" />

图1 Hololens 应用场景

#### 空间映射
借助微软特殊定制的全息处理单元（HPU），HoloLens 实现了对周边环境的快速扫描和空间匹配。这保证了 HoloLens 能够准确地在真实世界表面放置或展现全息图形内容，确保了核心的 AR 体验。

#### 场景匹配
HoloLens 设备能存储并识别环境信息，恢复和保持不同场景中的全息图像对象。当你离开当前房间再回来时，会发现原有放置的全息图像均会在正确的位置出现。

#### 自然交互
HoloLens 主要交互方式为凝视（Gaze）、语音（Voice Command）和手势（Gesture），这构成了 HoloLens 的基本输入要素。同时传统的键盘鼠标等设备也被支持，自然的交互方式更贴近人类习惯，提高了交互效率。

#### 通用应用
HoloLens 平台的操作系统为 Windows Holograpic，同样基于 Windows 10 定制。所以 Windows 10 UWP 通用应用程序可以顺利地在 HoloLens 上运行。这不仅降低了研发和迁移成本，也让开发效率大幅提升。

当然，说了很多 HoloLens 的特性和优点后，开发者版本也存在一些亟待解决的问题，比如视野较窄、凝视体验不佳、抗光线干扰弱和重量续航等。但瑕不掩瑜，HoloLens 带来了真正的混合现实体验，拥有着强烈的冲击感，未来将大有作为。

### 开发一个 HoloLens 应用
在了解 HoloLens 设备后，我们来试着开发一个简单的 HoloLens 应用，当然你也可以开发一个传统的 UWP 应用。这里我们则采用 Unity 引擎来构建应用，使用 Unity 开发是官方推荐的做法。

#### 开始之前
确保正确配置了开发环境，需安装以下工具和 SDK：

- Visual Studio 2015 Update 1及以上版本；

- Windows 10 SDK 10586及以上版本；

- HoloLens 模拟器，如图2；

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d60150447a.png" alt="图2 HoloLens模拟器" title="图2 HoloLens模拟器" />

图2 HoloLens 模拟器

- Unity HoloLens 技术预览版。

以上工具和 SDK 均可在微软官方网址获取，详细教程可以访问：<https://developer.microsoft.com/en-us/windows/holographic/install_the_tools> 。

#### 集成 HoloToolkit-Unity 项目
在创建了标准 Unity 项目之后，我们需要集成微软官方提供的 HoloToolkit-Unity 项目。HoloToolkit-Unity 是微软官方的开源项目，用于帮助开发者快速开发 HoloLens 应用，能够快速为项目集成基本输入、空间映射和场景匹配等特性。以下是此项目的结构和内容分析，如图3。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d6023c7c29.png" alt="图3 HoloToolkit-Unity项目结构 " title="图3 HoloToolkit-Unity项目结构 " />

图3 HoloToolkit-Unity 项目结构

Input 目录

- GazeManager.cs 用于快速集成凝视射线特性；

- GestureManager.cs 用于快速集成手势识别特性；

- KeywordManager.cs 用于快速集成语音命令特性；

- CursorManager.cs 用于快速集成可视化凝视组件。

Sharing 目录

- Sharing Prefab 组件用于快速集成场景共享特性。

SpatialMapping 目录

- SurfacePlane Prefab 组件用于描述和渲染真实世界表面；

- SpatialMapping Prefab 组件用于快速集成空间映射特性；

- RemoteMapping Prefab 组件用于快速集成远程空间映射信息导入特性；

SpatialSound 目录

- UAudioManager.cs 用于快速集成空间声音特性。

Utilities 目录

- Billboard.cs 用于实现跟随用户视线特性；

- Tagalong.cs 用于实现跟随用户移动特性；

- Main Camera Prefab 组件用于快速集成 HoloLens 标准主摄像机。

#### 构建场景
新建空白场景后，我们需要删除原有的 Main Camera 对象，同时从 HoloToolkit 目录中拖拽一个 Main Camera Prefab 组件到场景中，如图4，这样就集成了满足 HoloLens 需求的基本主摄像机。对于 HoloLens，将主摄像机渲染背景设为纯色，颜色设为 RGBA(0,0,0,0)。因为任何纯黑的颜色将会被 HoloLens 渲染为透明，以达到不遮挡现实世界的目的。此外，HoloLens 建议摄像机视角近距离为0.85，这个距离最符合真实人眼的体验。同时主摄像机位置必须重置为世界零点，即 xyz(0，0，0)，任何全息图像将会以此为原点在周边世界中绘制出来。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d60303bb5d.png" alt="图4 设置主摄像头" title="图4 设置主摄像头" />

图4 设置主摄像头

然后点击“Create Empty”创建一个空游戏对象，并将其命名为 Input，如图5。为 Input 对象添加核心脚本组件，分别为 GazeManager.cs、GestureManager.cs、HandsManager.cs 和 KeywordManager.cs。这样就集成了以上命令三大核心特性，对于凝视射线、手势识别和语音命令功能，均建议使用单例来进行管理，这样可以避免功能混乱。同时为凝视设置可视化的指针，可以提高用户的交互体验和效率。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d603cb4b19.png" alt="图5 集成输入组件" title="图5 集成输入组件" />

图5 集成输入组件

接下来集成可视化凝视组件，从HoloToolkit 目录下拖拽 CursorWithFeedback Prefab 组件到场景中，如图6。这样当凝视在全息对象时，其表面会出现可视化凝视组件。当用户手势被识别到时，会出现一个蓝色的手掌图像，能够贴心地告诉用户可以操作了。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d60488e04c.png" alt="图6 集成凝视组件 " title="图6 集成凝视组件 " />

图6 集成凝视组件 

创建一个Cube对象和一个新的 C# 脚本，命名为 HoloTest.cs。Cube 作为我们的全息图像主体，将它的 Transform 参数设为如图7所示。这样 Cube 的位置方便我们近距离观察其实际变化情况，你也可以根据自己偏好来放置它。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d6054063f9.png" alt="图7 设置Cube的Transform参数" title="图7 设置Cube的Transform参数" />

图7 设置 Cube 的 Transform 参数

HoloTest.cs 脚本的功能为随机更换对象的材质颜色，遵循 GestureManager.cs 中预设的 OnSelect 消息名称，HoloTest.cs 脚本中将会在 OnSelect 方法中实现此功能代码如下：

```
public void OnSelect()
    {
        //随机变换物体颜色
        gameObject.GetComponent<meshrenderer>().material.color = new Color(Random.Range(0, 255) / 255f, Random.Range(0, 255) / 255f, Random.Range(0, 255) / 255f);
    }</meshrenderer>
```

进入 Input 组件检视选项卡，为 KeywordManager.cs 组件配置语音命令。图8语音命令触发时将会执行相应的组件行为。本例中，当我说出“test”时，机会即会 Cube 的 OnSelect 方法，来随机改变 Cube 颜色。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d607941629.png" alt="图8 设定语音关键词行为" title="图8 设定语音关键词行为" />

图8 设定语音关键词行为

#### 编译项目
为了满足 HoloLens 的需求，我们需要在 Player Settings 里面开启 Virtual Reality Support，并在下拉列表中选中 Windows Holographic，如图9。只有这样 HoloLens 才会将此应用渲染为 3D 应用，这一点十分关键。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d608776b0c.png" alt="图9 添加HoloLens支持" title="图9 添加HoloLens支持" />

图9 添加 HoloLens 支持

同时从工具栏 Edit→Project Settings→Quality 选项卡中，将 UWP 平台默认画质设为 Fastest，如图10。这是为了降低性能开销，官方推荐帧率为60 fps。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d60929ae8a.png" alt="图10 设定默认画质" title="图10 设定默认画质" />

图10 设定默认画质

同时从工具栏 Edit→Project Settings→Quality 选项卡中，将 UWP 平台默认画质设为 Fastest，如图10。这是为了降低性能开销，官方推荐帧率为60 fps。

如图11，Build Settings 视图中选择目标平台为 Windows Store，SDK 为 Universal 10，点击 Build 按钮开始编译 UWP 项目。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d60a664973.png" alt="图11 编译Unity项目" title="图11 编译Unity项目" />

图11 编译 Unity 项目

#### 部署调试应用
使用 Visual Studio 打开编译后的 UWP 项目，在 Debug 选项上设置如图12所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d610563e31.png" alt="图12 设置Debug选项" title="图12 设置Debug选项" />

图12 设置 Debug 选项

连接 HoloLens 到 PC，完成 Build 和 Deploy 后，我们在 HoloLens 中打开此应用。实际效果如图13所示。当我使用手势点击 Cube 时，它会随机变化颜色；而当我说出语音命令“test”时，Cube 仍会正常的变换颜色，这完全符合我们的预期。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d611105354.png" alt="图13 实际效果图" title="图13 实际效果图" />

图13 实际效果图

### HoloLens 开发总结
使用 Unity 引擎开发 HoloLens 应用是非常容易的事情，大部分流程与开发 UWP 项目并无不同。但仍有不少需要注意的雷区和特殊要求，以下就是部分要注意的部分：

1. Main Camera 一定要按照官方要求配置，背景纯色且 RGBA 值为（0，0，0，0），这样才能避免遮挡现实内容；

2. Gaze 凝视特性需要我们使用 Raycast 来实现，注意处理射线未命中目标情形，默认凝视最远距离为15米，若是未击中物体，使用时可能会出现空引用异常；

3. 手势识别、拍照和语音命令等均需使用 Windows 特有 API，空间映射和场景匹配需要使用 HoloLens 特有 API；

4. 其他很多细节上的体验，例如可视化凝视组件、目标区域可视化指引组件等，使用它们来给用户提示，可以帮助用户理解应用操作方法，提高使用体验。

最后，AR/MR 技术独特的交互体验与开发特性，代表了未来自然交互的发展方向，相较于目前成熟的 VR 技术，它们具有更光明的发展前景和更广阔的用途。无论是微软还是 Magic Leap，它们无疑会是未来市场的引领者，而目前也是我们学习的黄金阶段，能够迎头赶上这波浪潮，对于相关从业者具有重要的意义。