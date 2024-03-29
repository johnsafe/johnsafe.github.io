## 移动直播连麦实现——A 端合成

文/张亚伟

>本文是移动直播连麦功能实现系列的第三篇，分享 A 端合成连麦音视频的流程及细节。此前我们介绍了连麦实现思路的整体情况，包括音视频合成的几种实现思路，A 主播及连麦的各参与角色定义等。以及使用 UpServer 服务器合成音视频数据的流程，包括 UpServer 合成、AB 主播端合成、时间戳生成及使用等，内容详情见《程序员》2016年10月刊和11月刊。

本内容包括 A 主播音视频数据合成，B 主播音视频数据合成，UpServer 转发音视频数据、传输协议介绍等小节。其中大量细节都是在第二篇《移动直播连麦实现——Server 端合成》中定义的，如视频基本参数定义、显示位置定义、音频合成算法、时间戳定义和使用说明等，为避免重复说明，此处直接使用，如看本文档时，遇到看不懂的细节，请参考《程序员》2016年11月刊。

当 A 端合成连麦的音视频数据流后，会把该数据流推送给 UpServer，再由 UpServer 推送给 DeliveryServer 集群，然后由 DeliveryServer 分发给移动直播的所有观众。DeliveryServer 集群及后续的推送工作不涉及音视频合成，与连麦无关，不属于本文关注的范围，下文不再描述，结构图也仅绘制到 DeliveryServer，望大家了解。

### A 主播视频

A 主播端视频合成处理较为复杂，分为三步，第一步是 A 主播接收所有 B 主播视频，并进行解码和缓存；第二步是 A 主播本地视频的采集，与远程视频进行合成、显示等；第三步是编码、打包、发送合成好的视频图像给 UpServer 服务器。

下面分别对这三个步骤进行详细说明。

**接收 B 主播视频**

A 主播负责视频合成时，需要先拿到所有 B 主播的视频数据，而 B 主播的视频都是由 UpServer 服务器转发给 A 端的，如图1所示。此时，所有 B 主播的视频数据都发给 UpServer，再由 UpServer 转发给 A 主播；针对 B 主播的视频数据，UpServer 不需要额外处理，按独立数据流发给 A 主播即可。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdc4e9b80f.png" alt="图1  A主播接收视频数据流结构图" title="图1  A主播接收视频数据流结构图" />

图1  A 主播接收视频数据流结构图

A 主播需要创建不同的逻辑对象，分别用于接收 B1、B2、B3 主播的视频数据，并在接收排序后按时间戳拼帧、解码，得到并保存每个 B 主播的视频图像。

**合成本地和远程视频**

在直播中，A 主播自身也要持续不断地采集视频，按照预定的时间间隔得到每帧本地视频后，与上一步得到的 B 主播视频进行合成；A 主播本地视频采集帧率是十分稳定的，无需考虑丢包、延迟等造成帧率不稳定的问题，故视频合成后的帧率就是A主播本地视频采集的帧率。

视频参数、显示位置预定义、视频尺寸的缩放、颜色空间的转换、视频图像数据的存储格式等都与第二篇 Server 端合成相同，这里不再说明。下面的 A 主播视频合成流程介绍中继续复用角色定义，B1、B2、B3 代表各 B 主播的图像缓存逻辑对象，A 代表 A 主播本地采集到的视频，合成后的视频图像仍然存储在 VideoMixer 中。

1. 首先读取 A 主播的视频数据作为合成底图，由于是本地采集，无需考虑读取失败情况。

2. 拷贝 A 图像上部的58行到 VideoMixer 中，行数计算公式是640-160\*3-1\*2-100=58。

3. 接着读取 B3 主播的视频数据进行合成，有成功和失败两种情况，如读取成功接第5步，如读取失败也分为两种情况，一是有上一帧，二是没有上一帧。如有上一帧也是执行第5步，如没有上一帧或不存在 B3 主播则见下一步。

4. 如读取 B3 视频失败且上一帧不存在，或者不存在该主播，则直接拷贝 A 的视频图像161（160+1）行到 VideoMixer 中。

5. 如读取 B3 视频数据成功或有上一帧，则把 B3 和 A 主播视频图像进行合成，这里需要逐行合成，每行要分成2段进行拷贝，第一段拷贝 A 视频数据到 VideoMixer 中，拷贝列数360-120=240；第二段拷贝 B3 视频数据到 VideoMixer 中，120列。

6. 如读取 B3 视频数据成功，合成之后还要备份 B3 的视频数据到指定位置，用于下次读取失败时复用上一帧；当 B3 主播停止连麦或较长时间卡顿后，还需要清除该数据缓存，避免引起歧义。

7. 接下来拷贝 B3、B2 之间间隔部分，即 A 的视频数据到 VideoMixer 中，1行。

8. 循环3-7步过程，把 B2、B2 与 B1 之间间隔（A 的1行）的视频数据拷贝到 VideoMixer 中；再循环一次，把 B1、B1 下部（A 的100行）的视频数据拷贝到 VideoMixer 中。

9.使用上述流程，就完成了“A+B1+B2+B3”视频图像合成，合成结果已存储在 VideoMixer 中。

A 主播本地的视频显示也要包括各 B 主播视频（A+B1+B2+B3），故使用合成好的 VideoMixer 绘制即可。

#### 发送合成视频

A 主播生成“A+B1+B2+B3”视频图像后，需要把该视频帧编码、打包，并发送给 UpServer 服务器，再由 UpServer 服务器转发该视频流给各 B 主播和 DeliveryServer，如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdc580a514.png" alt="图2  A主播发送视频数据流结构图" title="图2  A主播发送视频数据流结构图" />

图2  A 主播发送视频数据流结构图

DeliveryServer 接收并转发“A+B1+B2+B3”视频流，用于移动直播用户观看连麦视频，后续的发送流程与移动直播相同，不再赘述。各 B 主播接收“A+B1+B2+B3”视频流，用于显示 A 主播及其他 B 主播的视频，对于 B 主播视频合成和显示的细节，将在 B 主播视频合成小节中讲述。

总结，在移动直播连麦 A 端合成视频模式下，A 端需要接收和发送大量媒体数据，执行的任务繁重、实现的业务功能复杂，需要消耗更多的 CPU 资源，所以 A 主播的手机网络状态和 CPU 处理性能是最大瓶颈，这点一定要注意。如果可以使用硬件编码和解码，就尽量使用，毕竟就算是最好的手机设备，若手机使用过程中发热量太大，也对业务应用有一定影响。

### B 主播视频

B 主播视频处理过程与 A 主播相比要简单一些，主要分为发送、接收显示两个部分。发送部分也是采集、编码、打包、发送等，与其他合成模式下的流程一致，仅需注意下视频尺寸为160*120，毕竟视频尺寸小消耗的处理资源也少一些。B 主播视频数据流发送的结构图，可参考上一节的图1，这里不再重新绘制。

接下来说 B 主播接收视频及后续的合成流程，有两种方式，以 B1 主播为例说明如下：

- 第一种，UpServer 推送合成好的视频“A+B1+B2+B3”给 B1，与推送给普通观众的视频流相同，如上一节的图2所示。此时 B1 需要把本地采集到的视频合成到“A+B1+B2+B3”顶层上，替换掉原有的 B1 视频图像即可，从本地视频显示的实时性要求来看，这步替换是必须要做的。

- 第二种，UpServer 推送合成好的视频“A+B1+B2+B3”给 B1，同时 UpServer 也把 B2、B3 的视频数据独立推送给 B1，如图3所示。此时 B1 主播收到独立的3路视频数据流，需要把本地采集到的视频及 B2、B3 主播视频，合成到“A+B1+B2+B3”顶层上，替换掉 A 主播已经合成好的 B1、B2、B3 视频图像。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdc6501056.png" alt="图3  B1主播接收视频数据流结构图" title="图3  B1主播接收视频数据流结构图" />

图3  B1 主播接收视频数据流结构图

先描述下以上两种视频合成方式的细节。第一种方式与 UpServer 合成模式下的 B 主播视频合成流程完全相同，细节不再复述。而第二种方式 B1 的处理流程更为复杂一些，需要把“A+B1+B2+B3”、B2、B3 的视频数据都解码，然后按照音视频同步的顺序读取和合成，得到的结果是新的合成图像“A+B1（新）+B2（新）+B3（新）”，视频合成流程与 A 主播视频合成类似。

接着比较下两种视频合成方式的优缺点。第一种方式以前就建议过，从功能和业务角度来说，可以满足需要。而第二种方式的优缺点更明显一些，缺点是需要处理的内容更多了，包括 B2、B3 主播视频的解码、同步和合成等，需要消耗更多的系统资源；优点也有以下两点：

1. 与第一种方式相比，B2、B3 主播视频的实时性更好一些，毕竟使用 UpServer 直接发送的 B2、B3 视频数据进行合成，比使用 A 合成好的 B2、B3 视频数据，处理延迟和网络延迟占用的时间可以明显降低。

2. B2、B3 主播的音视频同步效果会更好一些。一般情况下 B1 需要独立接收 A、B2、B3 主播的音频数据，当 B1 播放 B2 的音频、视频时，由于两路数据流都有，所以同步播出的效果会比较理想；而使用方式一，B2 视频是由 A 合成好的，当 B1 主播播放 B2 的音频时，无法和 B2 视频进行同步。

处理延迟时间是指 A 合成 B2、B3 视频消耗的时间，包括解码、合成、编码，打包等；网络延迟时间是指 B2、B3 的视频数据，要先由 UpServer 要发给 A，A 完成视频合成后再发送回 UpServer 网络传输使用的时间。这两个时间加在一起，在处理比较好的情况下，估计也需要1秒以上，此处还不考虑网络比较差导致的丢包、网络中断导致的卡顿情况，毕竟网络质量差，连麦的体验效果也必然非常差。

使用方式一，A 主播合成的“A+B1+B2+B3”视频时间戳必然是 A 主播的，可以很好的和 A 主播音频进行同步，但不能和独立的 B2、B3 主播音频数据时间戳同步。由于延迟占用时间，A 主播合成的 B 主播视频必然比独立发送的 B2、B3 主播音频更晚到达 B1 主播，导致 B1 主播播放 B2、B3 音频时，与 B2、B3 视频出现不同步的可能性很大。

总结，以上两种方式，方式一的处理流程简单一些，消耗的资源也少，而方式二处理流程更为复杂，且消耗的资源也多；但在用户体验效果方面，方式二会更好一些，结论是如果非常重视用户体验，且消耗的资源可以承受则使用方式二，否则使用方式一。如果是第一次实现，建议先使用方式一实现，待收到更多效果反馈后，再确定是否改为方式二，毕竟方式二仅对 B 主播的用户体验改善有效，而与移动直播的众多接收用户体验效果无关。

### UpServer

这个小节是否需要，笔者也是反复进行了考虑，最终还是确定保留下来，为什么犹豫不决呢？原因是在 A 主播音视频合成模式下，UpServer 转发工作相当简单了，仅需要按设计要求，把每路音视频数据转发给相应的对象。且媒体数据转发的流程都已在各小节进行详尽介绍和绘图说明了，故感觉增加本小节的意义不大。

然而，如没有本小节也总感觉文章不够完整，毕竟 UpServer 是移动直播连麦的重要组件，其负责的转发媒体数据工作也是连麦实现的重要环节，故决定单独列出。

表1列出了 UpServer 转发各媒体数据的源、目标等，供大家整体参考。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdce252169.jpg" alt="表1  UpServer转发源目标说明" title="表1  UpServer转发源目标说明" />

表1  UpServer 转发源目标说明

### A 主播音频

A 主播音频与视频处理流程类似，先介绍其步骤，第一步是 A 主播要接收所有 B 主播音频，并进行解码、合成和播放；第二步是 A 主播本地音频的采集，与远程音频数据进行合成；第三步是编码、打包、发送合成好的音频数据给 UpServer 服务器。

需要注意的是第二步和第三步，当 A 主播负责合成音频时，需要生成两路音频数据流，都推送给 UpServer，一路是“A+B1+B2+B3”，用于移动直播用户收听连麦语音，另一路是 A 本地音频，用于 B1/B2/B3 主播收听 A 主播音频。这样做的原因在上一篇文章解释过，在用户体验上，B 主播肯定不希望听到自己的声音（类似回音），即 B1/B2/B3 主播都不能使用合成好的音频数据。

下面分别对这三个步骤进行说明。音频合成算法，仍使用上篇文档介绍过的直接相加算法，该算法简单高效，且适用范围还不错。

#### 接收、合成和播放 B 主播音频

A 主播负责音频合成时，需要先拿到所有 B 主播的音频数据，而 B 主播的音频数据都是由 UpServer 服务器转发给 A 的，结构图与第一节图1相同。此时，所有 B 主播的音频数据都发给 UpServer，再由 UpServer 转发给 A 主播；针对 B 主播的音频数据，UpServer 不需要进行额外处理，直接发给 A 主播即可。

A 主播需要创建不同的逻辑对象，分别用于接收 B1、B2、B3 主播的音频数据，并在接收后按包序号排序和按时间戳顺序解码，之后合成所有 B 主播音频数据。继续复用角色定义，各 B 主播音频缓存对象分别用 B1、B2、B3 代表，合成后的音频数据存储在 AudioMixerB 中。

B 主播音频数据合成的流程如下。

1. 先找到可以合成的音频数据，若没有就不必合成了。若只有一路音频数据，也无需合成直接拷贝其到 AudioMixerB 中即可。

2. 音频数据合成必须有两路以上，一般是先找 B1 主播再找 B2 主播，依次类推，直到找到可以合成的所有音频数据。

3. 通过循环遍历所有采样点，把各 B 主播的音频数据直接相加，结果放到 AudioMixerB 中，即完成了音频数据合成；直接相加算法需要对音频数据溢出进行保护，务必注意。

最后是播放 B 主播合成好的音频数据（B1+B2+B3），与一般的移动直播用户播放方法相同。

#### 合成本地和远程音频

在移动连麦直播中，A 主播自身也要持续不断地采集本地音频，之后与上面合成好的音频数据（B1+B2+B3）再次合成，供广大直播观众收听。合成方法与上面相同，结果存储在 AudioMixerA（A+B1+B2+B3）中。

#### 发送合成音频和独立音频

发送音频数据给 UpServer 是 A 主播音频处理的最后一步，要分别把 A 主播自己的音频数据，合成好的音频数据（A+B1+B2+B3），编码、打包、发送；即使用两个逻辑对象分别存储和维护独立的音频数据与合成的音频数据，如图4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdc7505257.png" alt="图4  A主播发送音频数据流结构图" title="图4  A主播发送音频数据流结构图" />

图4  A 主播发送音频数据流结构图

这里有需要特别注意的一点，与一般的移动直播情况不同，移动直播连麦 A 主播合成音频，在与 UpServer 建立媒体数据传输通道时，需要建立两个音频通道且可以同时发送数据。在 A 主播和 UpServer 之间建立两个音频通道时，交互信息一定要做好，使 UpServer 可以很容易的区分，哪路音频数据是发给 B 主播的，哪路音频数据是发给 DeliveryServer 的。

### B 主播音频

B 主播音频处理过程与 A 主播相比要简单一些，仍然是分为发送、接收播放两个部分。首先说 B 主播的音频发送，其也是由采集、编码、打包、发送等几个环节组成，以 B1 主播举例，其直接把音频数据推送给 UpServer 即可，与其他合成模式下的流程一致。

在 UpServer 转发 B1 主播的音频数据时需要进行广播，分别发送到 A 主播、B2、B3 主播处，如图5所示，只有这样做才能使每个主播合成声音时达到最理想的效果，具体的原因见以前分析。第二节讲 UpServer 推送 B1 主播视频时有两种方式，其第二种是把 B1 主播视频推分别送给 A 主播、B2、B3 主播，这种方式与 B1 主播音频数据推送的形式比较接近。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdc84c636f.png" alt="图5  B1主播发送音频数据流结构图" title="图5  B1主播发送音频数据流结构图" />

图5  B1 主播发送音频数据流结构图

接下来说 B 主播接收音频及合成流程，以 B1 主播举例，由于 UpServer 已向其转发了 A 主播、B2、B3 主播的音频数据，如图6所示，所以合成流程清晰明确。B1 主播音频合成与播放流程，可参考上一节的 A 主播合成所有 B 主播音频数据及播放流程，它们之间的差异很小，不再重复。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/583fdc8e6e253.png" alt="图6  B1主播接收音频数据流结构图" title="图6  B1主播接收音频数据流结构图" />

图6  B1 主播接收音频数据流结构图

### 传输协议

在移动直播连麦的过程中，A 主播、各 B 主播、UpServer 之间需要非常及时的媒体数据交互，故网络传输协议的选择非常重要，当前常用的有 RTP、RTMP、HLS 等，本节将对这三种协议进行比较分析，从而给出建议。

RTP 和 RTMP 都是设计用来进行实时媒体数据通信的网络协议，能很好地支持音视频传输和数据通信，而 HLS 协议则是把媒体数据进行切片后，使用 HTTP 渐进下载方法实现的，HLS 协议是准实时的。一般情况下，RTP 协议底层使用 UDP 包进行传输，RTMP 使用 TCP 或轮询 HTTP 连接进行传输，HLS 使用 HTTP（TCP）连接进行传输。

UDP、TCP（HTTP）协议之间的优缺点相信大家都已非常了解，故不再阐述，下面讨论下在移动直播连麦各终端之间，协议选择的一些建议。

首先排除 HLS 协议，该协议的传输实时性与切片大小直接相关，切片大延迟太长，无法满足连麦的实时性要求，而切片小维护成本高，得不偿失只能放弃了。

接着分析 RTMP 和 RTP 协议，第一步先比较网络资源消耗方面，RTMP 使用 TCP 连接传输媒体数据，为维护 TCP 连接，一般比 RTP 使用的 UDP 消耗网络资源更多一点，尤其当网络条件转差后，传输同样的数据消耗的资源跟多，延迟更大，所以网络质量比较差时，使用 UDP 收发数据包的 RTP 协议更有优势。

第二步比较复杂网络结构的适应性，在穿透防火墙和跨越复杂网络方面下，可以使用 HTTP 请求数据的 RTMP 协议更有优势；而使用 UDP 协议传输数据时可能受到防火墙阻止，或者异构网络的限制，各种特殊现象的处理需要增加很多额外工作。

第三步比较网络处理功能实现难度，由于 UDP 协议传输数据易丢包、乱序，为提高用户体验，需要使用缓冲池、补包、按序排列等技术手段进行保护，必然增加处理难度。TCP 协议保证数据不会丢失和乱序，最多也就是传输慢或网络连接断开，所以上层处理媒体数据时不需要考虑重传、排序等，难度明显下降。

另外，在媒体数据分发 CDN 层面，当前 RTMP 协议已得到了很好的支持，而 RTP 协议支持尚现不足，在外部资源复用方面，也是一个需要考虑的因素。

总结，在网络资源使用方面 UDP 包传输数据的适应能力更加宽广，带宽好或差都可以达到很好的实时性，但其开发难度也比使用 TCP（HTTP）更大。个人建议，如果追求极致效果，且有足够强大的研发团队，建议移动直播连麦时，各主播之间使用 RTP（UDP）协议实现媒体数据传输；如果开发力量一般且追求产品快速上线，建议使用 RTMP（TCP）协议开发。

### 后记

通过以上介绍，相信读者对移动直播连麦的A端合成实现细节已基本了解。针对 A 主播合成，笔者个人看法是支持的，是否使用还请自行决定，列举已知的特点如下。

- 复杂度：该模式下 A 主播需要把本地采集的音视频数据，与 UpServer 转发的音视频数据进行合成，实现复杂度较高，但考虑到连麦时 A 主播的业务逻辑本就复杂，复杂度的适当增加也是可接受的。

- 主播端性能：A 主播实现复杂度高，必然要消耗更多的系统资源，需要依赖比较好的硬件设备，支持硬件视频编解码的移动设备应当是前提条件；同时，直播软件的性能优化也是研发团队需要特别关注的。

- Server 性能：UpServer 仅实现按要求转发媒体数据功能，无需参与音视频的解码、合成、编码等，CPU 使用少，业务应用成本大幅度下降了。

- 时间戳：A 主播合成音视频数据，生成的媒体数据流都以 A 主播本地时间戳为准，故在远端不存在参考歧义，观众方更易实现音视频同步。