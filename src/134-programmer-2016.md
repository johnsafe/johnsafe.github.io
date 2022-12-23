## Cocos2d-x 性能优化技巧及原理总结

文/满硕泉

作为当今最流行的游戏引擎之一，Cocos2d-x 被游戏开发团队广泛使用，性能优化一直是困扰广大开发者的一个重要问题，本文从多个角度和基于引擎底层的原理分享 Cocos2d-x 性能优化技巧，探讨如何提升游戏整体性能。

### 纹理优化

#### 纹理压缩和纹理格式

游戏包体中，纹理图片占据很大部分的体积，而纹理图片过大，会影响游戏性能的三个方面：首先，造成包体过大；其次，导致占用内存空间过大，从而造成游戏崩溃；最后，在向 GPU 传递图片数据时，图片数据过大会增加内存带宽的占用，从而引起读入大量图片时的卡顿。针对纹理图片优化主要有两点：纹理图片压缩和纹理图片缓存。

传统的纹理压缩只是单纯压缩纹理图片的数据，在读入到游戏时，还要有一个解压缩的过程，这样其实只是减少了纹理图片占据包体的大小，并没有降低内存中纹理图片所占的大小，除此之外，还增加了一个解压缩的过程，从而会增加纹理图片读入内存的时间。Cocos2d-x 的渲染底层-OpenGL 提供了压缩纹理，可以支持压缩纹理图像数据的直接加载。OpenGL ES 2.0中，核心规范不定义任何压缩纹理图像数据。也就意味着，OpenGL ES 2.0核心简单地定义一个机制，可以加载压缩的纹理图像数据，但是没有定义任何压缩格式。因此，包括 Qualcomm、ARM、Imagination Technologies 和 NVIDIA 在内的许多供应商都提供了特定于硬件的纹理压缩扩展。这样，开发者必须在不同的平台和硬件上支持不同的纹理压缩格式。比如苹果的设备均采用 Powervr GPU 支持 PVR 格式的压缩格式。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579ffea719b05.png" alt="图1  TexturePacker支持的图片格式" title="图1  TexturePacker支持的图片格式" />

图1  TexturePacker 支持的图片格式

在 Cocos2d-x 中，我们采用 TexturePacker 进行图片的压缩和打包，我们一般会选择 PVR.CCZ 的格式，PVR.CCZ 其实就是 PVR 的 ZIP 压缩形式，程序读入 PVR.CCZ 时会先解压缩成 PVR，然后再传给 GPU。PVR 纹理支持 PVRTC 纹理压缩格式。它主要采用的是有损压缩，也就是在高对比度的情况下，图片会有些瑕疵，和 PVR.CCZ 不同的是，PVRTC 不需要解压缩，但一方面图片会有瑕疵；另一方面 Android 设备基本不支持 PVRTC，一般情况下，在不需要支持 Android 设备时，会在一些粒子效果中使用 PVRTC 格式的纹理图片，因为在高速运动中，图片的瑕疵不是那么明显。

RGB 和 RGBA 格式如何选择呢？RGB 即16位色，就是没有 Alpha 透明度通道的格式，图片去除 Alpha 通道可以减少图片文件的大小，但在实际中，由于传递到 OpenGL ES 时需要数据对齐，在内存中会出现 RGB 和 RGBA 图片大小一样的情况。在16位色中，RGBA565 可以获得最佳质量，总共有65536种颜色。RGBA 中，RGBA8888 和 RGBA4444 是比较常用的格式，区别就是颜色总量，即 RGBA4444 可以表示的颜色值比较少，我们可以调用 Texture2D 的 setDefaultAlphaPixelFormat 函数来设置默认的图片格式。根据不同的需求选择正确的图片格式，是性能优化关键步骤。

对于图片的处理优化是无止境的，还可以通过将一张大图拆分成小图的方式来减小图片的大小。另外对于非渐进的背景，可以使用九宫格图片来减小图片的大小。

#### 纹理缓存和异步加载

除了纹理文件的大小会造成包体、内存和带宽的浪费以外，加载纹理的时间也是一个优化的重点，在开发游戏时，进入一些比较复杂的场景，可能会造成游戏的卡顿。Cocos2d-x 提供了两种功能以解决图片载入卡顿问题：纹理缓存和异步加载。

顾名思义，纹理缓存就是将图片缓存在内存中，而不需要使用时再加载，使用 SpriteFrameCache 调用 addSpriteFramesWithFile 函数将图片读入内存，读入过程会完成图片的解压缩（当图片的格式需要解压时）、数据读入和数据传到 GPU 的过程，然后用一个 set 来维护在缓存中的图片名集合，当调用 addSpriteFramesWithFile 函数传入图片名称时，如果图片名已经在 set 中，那么就不会再进行图片载入的一系列操作，从而提高了游戏的运行效率。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579fff66acde5.png" alt="图2  addSpriteFramesWithFile的过程" title="图2  addSpriteFramesWithFile的过程" />

图2  addSpriteFramesWithFile 的过程

纹理缓存是把双刃剑，在提升效率的同时，缓存图片会占用内存，从而增加游戏内存管理压力，一般的做法是，在不同场景切换时，释放缓存图片，从而释放内存，保持游戏占用的内存一直处于安全范围。这种处理方式有一个缺点，就是当两个场景存在一些共用图片时，会存在先释放后读入同一张图片的情况，这样既没有节省内存，也浪费了运行效率。解决这个问题的方法是将图片分为两类，只在某个场景中使用的图片和公用图片，公用图片即在一个以上场景中使用的图片，比如在游戏 UI 中使用的按钮等图片，一般在很多场景中都会使用，这种图片就要“常驻”内存中，需要维护一个图片列表，每个场景需要的图片列表以及公用图片列表。在游戏启动时，加载公用图片列表，然后该列表中的图片会一直存贮在缓存中；进入某个场景时，会加载该场景的图片列表；离开场景时，删除这个场景图片列表中的图片，从而腾出内存空间加载其他场景需要的图片和资源。

在进行图片删除时，有个值得注意的问题：要真正删除图片本身才能释放内存。很多开发者误以为 SpriteFrameCache 中的 removeSpriteFramesFromFile 函数中传入 plist 的名字就会删除这个图片的数据文件，实际情况并非如此，这个操作只是解除了图片的引用，但是删除具体的图片，需要调用 TextureCache 的 removeUnusedTextures。其中先调用前面的函数，后调用这个函数才会起作用。值得注意的是，使用 ccb 可以帮开发者调用 removeSpriteFramesFromFile，另外 removeSpriteFramesFromFile 传递的 plist 的名字如果不存在，也会出问题，最好的办法是修改 removeSpriteFramesFromFile 函数，做一下容错，检查传递的 plist 的名字是否存在；另外需要注意的是，需要在前一帧调用 removeSpriteFramesFromFile，后一帧调用 removeUnusedTextures 和 dumpCachedTextureInfo，这样才会起作用，因为引用删除后，才会删除所引用的图片。

异步加载是利用多线程，在读入某些比较大的图片（或者 3D 游戏中的模型）时，发起读入图片的“指令”不等待图片读入后返回而是继续进行其他操作，这样做的好处是可以消除进入场景时玩家的卡顿感，提升游戏的流畅度。在 Cocos2d-x 中，异步加载时通过调用 Director 类中的 TextureCache 实例的 addImageAsync 函数完成。

### Cocos2d-x 3.0的渲染优化

#### 自动批处理和自动裁剪

Cocos2d-x 3.0对于渲染部分的代码进行了重构和优化，采用了命令模式的设计模式，当每个节点调用 onDraw 函数的时候不是直接调用 OpenGL ES 的绘制函数，而是提交一个绘制命令，因此给了绘制命令优化的机会，增加了自动批处理和自动裁剪功能。

批处理，是 Cocos2d-x 3.0之前的版本的 SpriteBathNode，对于使用相同纹理和相同混合方式纹理，它通过整批次处理的方式，减少 OpenGL ES 的调用，从而提升游戏的效率。在 Cocos2d-x 3.0之前，我们需要显式地将节点作为 SpriteBathNode 的子节点，这样做相对比较复杂，在 Cocos2d-x 3.0中提供了自动批处理 Auto-BatchNode 的方式。它的实现原理，简而言之，需要绘制的精灵先存放到队列里，然后由专门的逻辑来渲染。对于队列中的精灵，一个个取出来，若材质一样的话（相同纹理、相同混合函数、相同 shader），就放到一个批次里；如果发现不同的材质，则开始绘制之前连续的那些精灵（都在一个批次里）。然后继续取，继续判断材质。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579fffa45e7c9.png" alt="图3   Cocos2d-x 3.0渲染流程" title="图3   Cocos2d-x 3.0渲染流程" />

图3   Cocos2d-x 3.0渲染流程

需要注意的是，由于此实现原理，不是连续创建的精灵，即使使用相同的纹理，相同的混合方式和相同的 shader，也无法实现 Auto-BatchNode，所以当我们使用相同的纹理绘制不同精灵时，要连续创建，并且使用相同的混合方式和相同的 shader，这样就可以借助 Cocos2d-x 的 Auto-BatchNode 提升游戏的运行效率。

除了自动批处理以外，Cocos2d-x 3.0还提供了自动裁剪 Auto-Culling 的功能。自动裁剪其实是我们常用的一种优化技巧，就是不绘制不在屏幕范围内的节点，从而节约程序运行的时间，提升效率，该优化方式需要我们做的是节点的逻辑部分，当节点移出屏幕之外时，除了停止绘制外，停止该节点的逻辑部分也可以提升程序的运行效率。

性能上的优化要更多的结合游戏的需求本身，提升性能可以减小卡顿，并且降低功耗。还有一种降低功耗的方法：就是动态设置帧率，通过调用导演类的 setAnimationInterval 函数可以动态的设置帧率，将动画比较少的 UI 界面帧率设置为30，将战斗或者动画比较多的界面帧率设置为60，因此帧率为30的界面就可以降低功耗。

#### 优化声音文件

纹理之外，声音文件也会在游戏中占据一定内存，优化声音文件也可以帮助我们降低游戏的峰值内存。优化方式需要考虑三方面，首先，建议还是采用单一声道。这样可以把文件大小和内存使用都减少一半；第二，尽量采用低比特率来获得最好的音质效果，一般来说，96 kbps 到128 kbps 对于 mp3 文件来说够用了；第三，降低文件的比特率可以减小声音文件的大小。

### 总结

在移动游戏开发中，由于设备性能的限制，优化性能是项很重要的工作，一些团队往往在游戏开发的后期才会考虑这个问题，这不是一个正确的方式，性能的优化工作要从项目的第一天开始关注，并且要整个项目组共同关注，对于新增的需求，要考虑采用什么样的方法才能达到最高性能，并且考虑新功能引入的性能风险，才能保证游戏有较高的运行效率。  

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/57a0006f3be46.jpg" alt="" title="" />