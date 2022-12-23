## 移动直播连麦实现——Server 端合成

文/张亚伟

>本文是移动直播连麦实现系列的第二篇，分享由 Server 端（UpServer）执行音视频合成的流程及细节。此前我们介绍了移动直播连麦实现思路的整体情况，包括音视频合成的几种实现思路，参与连麦的客户端角色和服务器角色，内容详情见《程序员》2016年10月刊。

本文章内容包括有 UpServer 音视频合成、A 主播音视频合成、B 主播音视频合成，以及时间戳应用介绍等小节，首先解析较为复杂的视频合成，后
详解容易些的音频合成。

使用 Server 端（UpServer）合成音视频后，UpServer 把合成好的数据流推送给 DeliveryServer 集群，再由 DeliveryServer 分发给移动直播的所有观众。DeliveryServer 集群及后续的推送工作不涉及音视频合成，与连麦无关，不属于本文关注的范围，不再描述，结构图也仅绘制到 DeliveryServer，望大家了解。

### UpServer 视频合成

视频是由分辨率、帧率、颜色空间、显示位置等基本参数定义的，在视频合成处理前，这些参数都要确定。为方便说明 UpServer 视频合成流程，
参考当前移动直播连麦的常见情况，对视频参数预先定义如下。

- A 主播视频数据的高和宽为640*360（16:9），置于底层全屏显示。

- B 主播视频数据的高和宽均为160*120（4:3），显示位置都在顶层。

- 当有多个 B 主播视频时，B1/B2/B3 主播的显示位置在右侧依次排列，每个主播间隔1像素，下部边缘保留100像素。

- 在水平方向，所有B主播都靠右侧显示，不留空隙。

- 显示顺序上既可以 B1 在下部，也可以 B3 在下部，但很多情况下只有1个 B 主播连麦，故一般安排 B1 在下部显示，好处是对 A 主播视频显示的影响最小。

- A 主播、B 主播的视频帧率都是每秒20帧，颜色空间都是 I420。

以上参数不仅在 UpServer 合成视频时需要使用，在 A 主播、各 B 主播视频合成时也需要使用。

如 A 主播、B 主播分辨率不符合上述定义，则需要先进行适当的缩放；如颜色空间不一致，也需要先进行转换。注意，视频数据的高和宽与显示的高和宽是不同的，在纵横比相同时，通过 OpenGL 自身的缩放即可实现视频图像的绘制，针对特殊屏幕分辨率的适配也有额外工作量。

视频帧率有差异，为简化操作无需插帧，可按 A 主播的视频帧率进行合成，即 B1/B2/B3 的视频都合成到 A 主播的视频图像上，A 主播的视频帧率就是合成后的视频帧率了。以 A 主播视频图像为底图进行合并，若 A 主播没有视频数据则合成暂停，若某个 B 主播没有新的视频数据，则仍然复用上一帧，从效果上看就是该 B 主播视频卡住了。

视频图像在内存中都是按行存储的，有左下角或左上角为起点两种情况，即常说的左底结构和左顶结构，具体遇到的是哪种结构试试便知。同时还要需要注意，颜色空间 I420中 YUV 是分成多个平面存储的，在合成图像时每个平面要单独处理，Y 平面和 UV 平面的行数、列数也是不同的。

以下我们的举例中，视频图像存储以左上角为起点（左顶结构），图像缓存复用角色定义，分别用 A、B1、B2、B3 代表，合成后的视频图像存储在 VideoMixer 中。当 UpServer 分别收到 A、B1、B2、B3 的视频数据流，并解码得到每一帧图像后就可以进行视频合成了，如图1所示。


![enter image description here](http://images.gitbook.cn/1be4b6b0-3cae-11e8-beb4-25e84190852b)

图1 UpServer 合成所有主播视频数据结构图

UpServer 合成视频的基本流程描述如下。

1. 首先根据音视频同步情况，读取 A 主播的视频数据作为合成底图，有成功和失败两种情况，如读取成功接第4步，失败则见第2步。

2. 当读取 A 主播视频图像数据失败时，仍然要按音视频同步情况读取所有 B 主播的视频数据，若读取成功则需要丢弃该视频帧，避免后续合成时使用该帧造成的不同步现象。

3. 当所有 B 主播视频数据都读取完成后，本轮视频合成就以失败结束了，视频合成依赖底图，无 A 主播视频就无法进行合成。

4. 拷贝 A 图像上部的58行到 VidoeMixer 中，行数计算公式是640-160\*3-1\*2-100=58，各项数值的意义请参考参数定义，大家应该已理解就不单独解释了。

5. 接着读取 B3 主播的视频数据进行合成，也有成功和失败两种情况，如成功接第7步，失败也分为两种情况，一是有上一帧，二是没有。如有也是执行第7步，如没有或不存在 B3 主播则见第6步。

6. 如读取 B3 视频失败且上一帧不存在，或者不存在该主播，则直接拷贝 A 的视频图像161（160+1）行到 VidoeMixer 中。

7. 如读取 B3 视频数据成功或有上一帧，则把 B3 和 A 主播视频图像按行进行合成，每行要分成2段进行拷贝，第一段拷贝 A 视频数据到 VidoeMixer 中，拷贝列数360-120=240；第二段拷贝 B3 视频数据到 VidoeMixer 中，120列。

8. 如读取 B3 视频数据成功，合成之后还要备份 B3 的视频数据到指定位置，用于下次读取失败时复用上一帧；当 B3 主播停止连麦或较长时间卡顿后，还需要清除该数据缓存，避免引起歧义。

9. 接下来拷贝 B3、B2 之间间隔部分，即 A 的视频数据到 VidoeMixer 中，1行。

10. 循环5-9步过程，把 B2、B2 与 B1 间隔（A 的1行）的视频数据拷贝到 VidoeMixer 中；再循环一次，把 B1、B1 下部（A 的100行）的视频数据拷贝到 VidoeMixer 中。

11. 使用上述流程，就完成了“A+B1+B2+B3”视频图像合成，合成结果已存储在 VidoeMixer 中。生成“A+B1+B2+B3”视频图像后，UpServer 需要把该视频数据流编码、打包，并发送给 DeliveryServer，用于移动直播用户观看连麦视频。

A 主播也需要观看其他 B 主播的视频，但是否使用“A+B1+B2+B3”的视频图像需要讨论，方法不同复杂度也不尽相同，具体将在 A 主播视频合成小节中说明。每个 B 主播也要显示 A 主播和其他 B 主播视频，个人认为使用“A+B1+B2+B3”的视频数据流就可以满足需求，具体细节在 B 主播视频合成小节中讲述。

### A 主播视频

A 主播的视频数据处理分为发送、接收、显示三个部分，发送环节包括采集、编码、打包、发送，接收环节包括接收、按时间戳同步、解包、解码，显示环节包括本地视频图像和远端视频图像的合成及显示。

发送部分，由 UpServer 负责合成视频时，A 主播需要把采集到的本地视频编码，并按指定封装格式打包，然后发送给 UpServer，流程结构如图1所示。该部分与直播连麦关系不大，与单一主播直播时流程一致。

A 主播接收部分，如图2所示，先接收 UpServer 推送的 B 主播视频数据，然后按时间戳同步读取、解包、解码。UpServer 为 A 主播推送的视频数据内容可以有以下三种，若数据内容不同，合成显示部分的处理流程也不同。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581841c280aad.png" alt="图2 UpServer合成，A主播接收视频结构图" title="图2 UpServer合成，A主播接收视频结构图" />

图2 UpServer 合成，A 主播接收视频结构图

- 第一种，UpServer 推送合成好的所有视频数据“A+B1+B2+B3”，与推送给普通观众的视频相同；

- 第二种，B1/B2/B3 视频数据流独立推送，即 UpServer 直接转发各 B 主播的视频给 A 主播；

- 第三种，UpServer 把 B1/B2/B3 视频按尺寸和显示位置合成为一路视频数据流，并把该路视频数据流推送给 A 主播。

下面分享 A 主播的合成显示部分，结合上面讲到的 A 主播接收的3种视频数据流形式，分别描述和比较它们的优劣。

先简单介绍个人不建议的形式，由 UpServer 按视频尺寸和显示位置把多个 B 主播的视频合成一路视频流，该方式的缺点是 UpServer 的压力显著增加。UpServer 已为每个连麦进行了视频合成工作，再为 A 主播另外合成一路，在视频合成和编码发送方面，承担了更多的压力，本身 Server 端合成最大的瓶颈就是 UpServer 服务器的处理能力，故任何增加其负担的行为都应该被摒弃。同时，该方式下 A 主播自身的解码、合成流程也没有什么变化，CPU 使用基本未减少，故不建议采用。

接着介绍 UpServer 推送合成好的所有视频“A+B1+B2+B3”给 A 主播的形式，此时 UpServer 仅增加了一路数据流推送，而没有更多的消耗，属于可以接受的方式。但该方式对 A 主播的图像合成压力比较大，为保证本地视频的实时性，底图的 A 视频图像必须使用刚刚采集的，所以合成图像前需要先抠图。具体步骤如下：

1. 把“A+B1+B2+B3”中的 A 扣掉，得到 B1、B2、B3 独立的3个视频；由于视频是按行存储的，所以获取独立视频的过程也只能是循环遍历每行，得到所需要的视频数据。

2. 使用最新采集到的 A 主播视频，合并上面得到的 B1、B2、B3 视频，按第一节讲解的视频图像合成再来一次，得到 “新 A+B1+B2+B3”。

该方式下，后续的显示流程差异不大，通过 OpenGL 把“新 A+B1+B2+B3”绘制到 UI 即可。

最后 UpServer 直接转发各 B 主播视频给 A 主播的方式，该方法的问题是推送三路数据流时，A 主播的网络丢包（接收丢包）、时间戳同步控制更麻烦，但在解码、合成方面则问题不大，且 UpServer 服务器增加的压力也非常微小。具体实现流程可以参考第一节的视频图像合成介绍。

总结，在 UpServer 推送 B 主播视频数据时，比较后建议选择第一种或第二种方法，特别是 UpServer 直接转发各 B 主播视频给 A 主播的方式（第二种），原因是第一种推送合成好的所有视频“A+B1+B2+B3”给 A 主播的形式有个瑕疵——如 B2 主播连麦一段时间后停止了，UpServer 和 A 主播收到该消息的时间是有先后差异的，会出现两种情况：

- UpServer 停止合成 B2 视频，但 A 主播仍然合成和显示；A 主播抠取 B2 视频时，得到的是 A 主播的图像，这样在 A 主播合成 B2 区域时，使用的就是 A 主播自身的历史图像了。

- UpServer 未停止绘制 B2 视频，但 A 主播不再绘制，则 A 主播会看不到 B2 的最后几帧视频。

以上两种情况，是由于消息接收和处理时间差异造成的，故会非常短暂，仅属于瑕疵。使用 UpServer 直接转发各 B 主播视频给 A 主播的方式，由于每路数据流是独立的，故不存在该瑕疵。

### B 主播视频

B 主播视频的处理过程主要分为发送、接收显示两个部分。发送部分也是采集、编码、打包、发送，与 A 主播流程一致。在接收显示方面与 A 主播相比，B 主播视频显示在合成好的所有视频顶层，故处理过程相对简单一些。以 B1 主播举例，有两种方式：

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581841d1528c0.png" alt="图3 UpServer合成，B1主播接收视频结构图" title="图3 UpServer合成，B1主播接收视频结构图" />

图3 UpServer 合成，B1 主播接收视频结构图

- 第一种，UpServer 推送合成好的所有视频“A+B1+B2+B3”，与推送给普通观众的视频流相同，如图3所示；

- 第二种，A 主播、B2、B3 主播视频数据流独立推送，即 UpServer 直接转发其他主播的视频给B1主播。

先说排除的方法，A 主播、B2、B3 主播视频数据流独立推送，该方法没有明显优点，若音频也是独立发送，则音视频同步方面可能稍好；缺点是 B1 主播需要接收3路视频数据流，且都要按时间戳同步读取数据、解码和合成图像，故 B1 主播消耗的 CPU 更高一些。

其次，UpServer 推送合成好的所有视频“A+B1+B2+B3”给 B1 的方法，该方法 B1 主播仅需读取一路数据和解码，之后数据合成也仅是把 B1 位置的远端视频替换为本地采集的，操作简单易行；在该方法下，合成视频的音视频同步可参考 A 主播音频完成，细节详见后续小节。

综上所述，建议选择 UpServer 推送合成好的所有视频数据流给 B1 主播的方法，实现 B1 观看其他连麦主播视频。B2、B3 主播的实现与 B1 主播相同。

### UpServer 音频合成

音频也是由一些基本参数进行定义的，包括采样频率、采样点占位数、通道数、每帧长度等。为方便说明 UpServer 音频合成流程，参考移动直播连麦的常见情况，对以上参数进行定义：采样频率是 48KHz，采样点占位数是16位（2字节），通道数是2（立体声），每帧长度为8192字节（便于编码），存储顺序为左右声道逐采样点交错。

音频数据流合成处理与视频相比简单一些，不区分主播类型都按以上参数采集、合成和播放，即所有主播都是一致的。

音频合成前，先介绍多人语音数据合成算法，大家知道音频合成算法有很多，如直接相加、取最大值、线性叠加后求平均、归一化混音（自适应加权混音算法）等，不同算法实现复杂度和特性各不相同，适用的场景也有很大差异。这里仅介绍较简单的音频合成算法：直接相加算法和取最大值算法，其他的算法介绍超出了本文讲解范围，请读者自行学习。

直接相加，顾名思义，是把两路音频数据直接相加到一起，由于两路数据的采样点都是16位，结果也是16位的，所以相加有数据溢出的可能，保护方法是判断数据溢出则把结果设置为最大值。该算法的缺点是容易爆音，优点是算法简单，由于数据合成导致的各路声音损失非常小。

取最大值，两路数据合成时逐采样点获取最大值，保存到合成结果中。由于数值较小的采样点被丢弃了，故该算法的缺点是声音数据失真较多；优点是算法简单，不存在溢出导致爆音的情况。

以上两种算法既适用于两路声音合成的情况，也适用于多路声音合成的情况。以下在音频数据合成时，我们默认采用直接相加算法。

各主播音频缓存复用角色定义，分别用 A、B1、B2、B3 代表，合成后的音频数据存储在 AudioMixer 中。当 UpServer 分别收到 A、B1、B2、B3 的音频数据流并解码得到每一帧数据后，就可以进行音频合成了，如图4所示。音频合成的基本流程如下。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581841e0630eb.png" alt="图4 UpServer合成所有主播音频数据结构图" title="图4 UpServer合成所有主播音频数据结构图" />

图4 UpServer 合成所有主播音频数据结构图

1. 先找到可以合成的音频数据，若没有就不必合成了。若只有一路音频数据，也无需合成直接拷贝其到 AudioMixer 中即可。

2. 音频数据合成必须有两路以上，一般是先找 A 主播再找 B1 主播，依次类推，直到找到可以合成的所有音频数据。

3. 通过循环遍历所有采样点，把各主播的音频数据直接相加，结果放到 AudioMixer 中，即完成了音频数据合成；直接相加算法需要对音频数据溢出进行保护，务必注意。

若所有主播都有声音数据，UpServer 音频合成后就得到了“A+B1+B2+B3”的音频数据，之后需要把该音频数据编码、打包，并发送给 DeliveryServer 用于移动直播的用户收听声音。

A 主播也需要听到其他 B 主播的声音，但不能使用“A+B1+B2+B3”的音频数据，原因在于用户体验上，A 主播不希望听到自己的声音（类似回音），B 主播也存在类似的问题，即所有主播都不能使用上面合成好的音频数据。

声音一旦合成，再想去除异常困难，故从“A+B1+B2+B3”的音频数据中去除 A 主播的声音，不进行尝试了。

为解决 A 主播、B 主播之间收听彼此声音问题，可以使用 UpServer 合成指定音频数据流方法，也可以使用向每个主播独立发送音频数据，由各主播自己合成的方法。

### A 主播音频

下面描述下 A 主播音频的处理过程，由于 A 主播采集的声音，不必和其收听的远端声音（B1/B2/B3）进行混合，所以其处理过程与视频差异很大。

采集方面，A 主播的音频采集流程相对简单，本地声音采集、编码、打包、发送给 UpServer 服务器即可，与移动直播是相同的。

移动直播连麦才需要的音频播放部分，接收音频数据流、解码、播放，是否需要合成取决于 UpServer 发送的数据流方式；大部分都是移动直播普通用户使用的环节；其中仅音频数据合成是连麦的特殊环节，这里详细讲解下。

A 主播是否需要合成取决于 UpServer 发送的数据流方式，故先介绍 UpServer 推送音频数据流的可能方式：

- 方式一，合成的音频数据，即由 UpServer 合成“B1+B2+B3”的声音数据发送给 A 主播，该方式下 A 主播不需要合成直接播放即可；

- 方式二，独立的音频数据流，UpServer 仅转发 B1/B2/B3 的音频数据，不进行合成，此时 A 主播需要合成 B1/B2/B3 的音频数据再进行播放，如图5所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581841ed38c1e.png" alt="图5 UpServer合成，A主播接收音频结构图" title="图5 UpServer合成，A主播接收音频结构图" />

图5 UpServer 合成，A 主播接收音频结构图

方式一与移动直播用户播放时流程相同，A 主播处理非常简单，但由于 UpServer 增加了资源消耗而不建议使用。在多个 B 主播连麦情况下，UpServer 为 A 主播合成“B1+B2+B3”的音频数据，为 B1 主播合成“A+B2+B3”的音频数据，为 B2 主播合成“A+B1+B3”的音频数据，为 B3 主播合成“A+B1+B2”的音频数据，需要多次合成、编码和打包，复杂度上升很多，感觉得不偿失故不再介绍了。

方式二 UpServer 仅透明转发 B1/B2/B3 的音频数据，A 主播需要对各 B 主播音频数据进行合成，合成算法复用上一节讲述的直接相加算法，合成流程也与 UpServer 合成音频数据的流程相同，这里不再赘述。该方式下 UpServer 的 CPU 消耗不高，但网络带宽要多用一些，考虑到音频数据流使用网络流量较低，该缺点属于可接受。

它们之间的差异，是把服务器端的合成工作，转移到主播端来做，从而降低服务器的资源使用，是个人推荐的方式。

### B 主播音频

在音频方面，不存在类似视频显示的主从关系，所有 A 主播和 B 主播都是平等的，故 B 主播音频的处理流程和逻辑，与 A 主播是基本一致的，差异是每个人需要接收的数据流不同，B1 主播需要“A/B2/B3”的音频数据，B2 主播需要“A/B1/B3”的音频数据，B3 主播需要“A/B1/B2”的音频数据。

实现细节请参考上一节 A 主播音频合成的介绍。

### 时间戳

为保证接收端音视频播放时达到最好的同步效果，在使用媒体数据传输协议打包时，都需要封装上本地的毫秒级时间作为时间戳（Timestamp）。

时间戳有相对时间和绝对时间两种方式，相对时间采用4字节存储，无符号类型在50多天会发生归零和重新累计，应用时需要对这点进行保护；绝对时间采用8字节存储，不必担心归零情况，但要多使用一点网络带宽。两种方式差异不大，都可以实现音视频同步功能。

在移动直播的远端音视频播放时，时间戳具有以下几个作用：

1. 为避免网络抖动产生的影响，需要对远端媒体数据先缓存后播放，而缓存数据的时间长度可以通过计算远端时间戳的差值得到。

2. 音视频是否可播放的判断条件，当缓存数据时间长度达到预期，或是本地时间差值和远端时间差值达到预期，就表明可以播放音视频数据了。

3. 音视频同步播放的判断条件，如以音频为基准，当音频可以播放了则比较音频时间戳和视频时间戳的偏差，在允许范围内也就该播放视频了。

由于 UpServer 需要先合成音视频数据，后按照媒体传输协议重新封装，故也需要再次打时间戳，而使用具体时间作为时间戳有些复杂。UpServer 使用的时间戳建议有两种，分别是复用主播端时间戳，或是使用本地时间，它们各有优缺点：

- 本地时间，直接使用本地的毫秒级时间作为时间戳，优点是实现简单，直接获取本地时间使用即可，缺点有两个：

1.  音视频数据合成、编码等会占用大量时间，如视频合成、音频不合成，由于处理速度差异，可能导致接收端音视频不同步。

2.  当不需要合成数据时也需要再次修改时间戳，如音频数据仅转发时；此时网络抖动丢包等对修改时间戳可能产生严重影响。

- 主播时间，复用主播媒体包中封装的时间戳，优点是合成编码等处理占用的时间，不会影响接收方的音视频同步播放，缺点也有两个：

1.  时间戳连续性无法保证，当音频合成时如使用 A 主播时间戳，但在 A 主播卡顿或无数据时则无法连续使用，此时其他 B 主播的声音仍然在不断合成中，还需要持续生成时间戳。

2.  同一主播的音视频，有时需要合成数据（合成视频），有时不需要合成数据（独立发送音频），此时被合成的视频由于复用了其他主播的时间戳，与未合成的音频，两个时间戳之间没有强依赖关系了，可能造成播放时不同步。

根据以上分析，结合使用 UpServer 合成音视频数据流的情况，整理和总结了推荐的媒体数据传输内容及时间戳的使用情况，如表1。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/5818430501880.jpg" alt="表1 推荐方案" title="表1 推荐方案" />

表1 推荐方案

表中有些是由 UpServer 负责合成，如“A+B1+B2+B3”，有些是 UpServer 透明转发的，如“B1/B2/B3”。在选择时间戳类型时存在不少困难，综合考虑后确定两种类型一起用，见时间戳列内的描述，如此选择的原因如下：

- 当接收方为 DeliveryServer 时，各主播都可能短时间没有声音数据，此时声音作为音视频同步的基准，必须要连续打时间戳，使用 UpServer 本地时间是最合理的；为了音视频同步，视频也就必须使用 UpServer 本地时间了。

- 当接收方是 A 主播时，推荐的形式是把各 B 主播的音视频数据直接推送给 A，此时复用原时间戳是最合理和简单的；

- 当接收方是 B 主播时，推荐的形式是视频合成、音频独立；音频独立不修改时间戳比较简单，而视频合成后只能依赖 A 主播视频，若选择 UpServer 本地时间为时间戳后，音视频无法同步，由于各 B 主播也要收听 A 主播声音，故视频选择复用 A 主播时间戳最为恰当。

按上述建议，视频合成数据“A+B1+B2+B3”，推送给不同终端时使用的时间戳不同，给 DeliveryServer 的视频数据使用 UpServer 本地时间打时间戳，推送给各 B 主播时复用 A 主播时间戳，这样是否是最合理的选择？大家可以深入研究。

### 后记

针对 Server 端合成，笔者个人的看法是持保留态度，是否使用还请自行决定，列举已知的劣势如下。

- Server 性能：UpServer 需要实现音视频的解码、合成、编码，CPU 使用增加较多，业务应用成本高。

- 复杂度：为了在主播、用户侧都达到最好的用户体验，各终端对数据流的要求不同，只能每个类型终端逐个适配，服务器和主播端的实现复杂度都很高。

- 时间戳：在不同数据流情况下，为实现音视频同步，不同终端需要的时间戳也是不同的，时间戳的产生和使用必须仔细推敲和详尽测试。

- 主播端性能：主播端为显示视频和播放声音也需要合成音视频，该工作要占用不少 CPU 资源；在 A 主播的性能和复杂度方面，与 A 主播合成相比优势微小。