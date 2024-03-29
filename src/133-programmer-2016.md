## IM 技术在多应用场景下的实现及性能调优：iOS 视角

文/陈宜龙

>IM 已经成为当下 App 的必备模块，在不同垂直领域，技术实现不尽相同。究竟该如何选型？技术实现过程中，又该如何进行性能调优？本篇文章分为应用场景、技术实现细节、针对移动网络特点的性能调优三个部分，具体讲解 IM 即时通讯技术在社交、直播、红包等不同场景下的技术实现与性能调优。

需要注意，本文中所涉及到的所有 iOS 相关代码，均已100%开源（不存在 framework ），便于学习参考；本文侧重移动端的设计与实现，会展开讲，服务端仅仅属于概述，不展开；本文还将为大家在设计或改造优化 IM 模块，提供一些参考。

### 大规模即时通讯的技术难点

首先，思考几个问题：

- 如何在移动网络环境下优化电量、流量及长连接的健壮性？现在移动网络有 2G、3G、4G 各种制式，且随时可能切换和中断，移动网络优化可以说是面向移动服务的共同问题。

- 如何确保 IM 系统的整体安全？因为用户的消息是个人隐私，所以要从多个层面来保证。

- 如何降低开发者集成门槛？

- 如何应对新 iOS 生态下的政策并结合新技术：比如 HTTP/2、IPv6、新的 APNs 协议等。

### 应用场景

IM 服务的最大价值在于什么？可复用的长连接。一切高实时性的场景，都适合使用 IM 来做，比如：视频会议、聊天、私信、弹幕、抽奖、互动游戏、协同编辑、股票基金实时报价、体育实况更新；基于位置的应用（Uber、滴滴司机位置）、在线教育、智能家居等。

接下来，我们会挑一些典型场景进行介绍，并分析其中具体的技术细节。

#### IM 发展史

IM 基本的发展历程是：轮询、长轮询、长连接。下面挑选一些代表性的技术进行介绍：

- 一般的网络请求：一问一答，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/5818464722389.jpg" alt="图1 正常请求" title="图1 正常请求" />

图1 正常请求

- 轮询：频繁的一问一答，见图2。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/5818465d217b2.jpg" alt="图2 轮询" title="图2 轮询" />

图2 轮询

- 长轮询：耐心地一问一答，曾被 Facebook 早期版本采纳，见图3。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184665ae1a9.jpg" alt="图3 长轮询" title="图3 长轮询" />

图3 长轮询

一种轮询方式是否为长轮询，是根据服务端的处理方式来决定的，与客户端没有关系。

短轮询很容易理解，那么，什么叫长轮询？与短轮询有何区别？长轮询和短轮询最大的区别是，短轮询去服务端查询时，不管服务端有没有变化，服务器就立即返回结果了。而在长轮询中，服务器如果检测到库存量没有变化话，将会把当前请求挂起一段时间（这个时间也叫作超时时间，一般是几十秒）。在这个时间内，服务器会检测库存量有没有变化，变化就立即返回，否则就等到超时为止。

我们可以看到，发展历史是这样：从长短轮询与长短连接，使用 WebSocket 来替代 HTTP。长短轮询到长短连接的区别主要有：

1. 概念范畴不同：长短轮询是应用层概念，长短连接是传输层概念；

2. 协商方式不同：一个 TCP 连接是否为长连接，是通过设置 HTTP 的 Connection Header 来决定的，且需要两边都设置才有效。

3. 实现方式不同：连接的长短是通过协议来规定和实现的。而轮询的长短，是服务器通过编程的方式手动挂起请求实现的。

在移动端上长连接是趋势，其最大的特点是节省 Header。对比轮询与 WebSocket 所花费的 Header 流量即可窥其一二。

假设 Header 是871字节，我们以相同的频率10 W/s 去做网络请求，对比轮询与 WebSocket 所花费的 Header 流量。其中，Header 包括请求和响应头信息。出于兼容性考虑，一般建立 WebSocket 连接也采用 HTTP 请求的方式，从这个角度讲，无论请求如何频繁，都只需要一个 Header。

并且，WebSocket 的数据传输是 Frame（帧）形式传输的，更加高效，对比轮询的2个 Header，这里只有一个 Header 和一个 Frame。而 WebSocket 的 Frame 仅仅用2个字节就代替了轮询的871字节！

相同的每秒客户端轮询的次数，当次数高达10 W/s 的高频率时，Polling 轮询需要消耗665 Mbps，而 WebSocket 仅仅只花费了1.526 Mbps，将近435倍，如图4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184672c3445.jpg" alt="图4 轮询与Websocket的花费的流量对比：435倍" title="图4 轮询与Websocket的花费的流量对比：435倍" />

图4 轮询与 Websocket 的花费的流量对比：435倍

数据参考：

1. HTML5 WebSocket: A Quantum Leap inScalability for the Web

2. 《微信、QQ 这类 IM App 怎么做——谈谈 WebSocket》

接下来，探讨下长连接实现方式的协议选择。

#### 大家都在使用什么技术？
笔者最近做了两个 IM 问卷，累计产生了900多条的投票数据，见图5、图6。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184680b63dd.jpg" alt="图5 使用何种协议" title="图5 使用何种协议" />

图5 使用何种协议

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/5818468fa9630.jpg" alt="图6 使用协议投票结果" title="图6 使用协议投票结果" />

图6 使用协议投票结果

注：本数据只能反映出 IM 技术在 iOS 领域的使用情况，并不能反映出整个 IT 行业的情况。

下文会对投票结果进行下分析。

**协议如何选择**？

IM 协议选择原则一般是：易于拓展，方便覆盖各种业务逻辑，同时又比较节约流量。后一点需求在移动端 IM 上尤其重要。常见的协议有：XMPP、SIP、MQTT、私有协议。我们这里只关注前三名，见表1。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/5818469b3fd99.jpg" alt="表1 XMPP、SIP、MQTT优劣对比" title="表1 XMPP、SIP、MQTT优劣对比" />

表1 XMPP、SIP、MQTT 优劣对比

一个好协议需要满足如下条件：高效、简洁、可读性好、节约流量、易于拓展，同时又能匹配当前团队的技术堆栈。基于以上原则，我们可以得出：如果团队小，在 IM 上技术积累不够可以考虑使用 XMPP 或者 MQTT+HTTP 短连接的实现。反之可以考虑自己设计和实现私有协议，这里建议团队有计划地迁移到私有协议上。

在此特别提一下排名第二的 WebSocket，区别于上面的聊天协议，这是一个传输通讯协议，那为什么会有这么多人在即时通讯领域运用了这一协议？除了上文说的长连接特性外，这个协议 Web 原生支持，有很多第三方语言实现，可以搭配 XMPP、MQTT 等多种聊天协议进行使用，被广泛地应用于即时通讯领域。

**社交场景**

在当前社交场景下，最大的特点在于：模式成熟，界面类似。我们团队专门为社交场景开发了一款开源组件——ChatKit-OC（项目地址：https://github.com/leancloud/ChatKit-OC），Star 数1000+。ChatKit-OC 在协议选择上使用的是 WebSocket搭配私有聊天协议的方式，在数据传输上选择的是 Protobuf（Google Prot ocol Buffer）搭配 JSON 的方式。

**直播场景**

在此分享一个演示如何为直播集成 IM 的开源直播 Demo：LiveKit-iOS（项目地址：https://github.com/leancloud/LeanCloudLiveKit-iOS）。LiveKit 相较社交场景的特点有：无人数限制的聊天室、自定义消息、打赏机制的服务端配合。

#### 数据自动更新场景

- 打车应用场景（Uber、滴滴等 App 移动小车）；

- 朋友圈状态的实时更新，朋友圈自己发送的消息无需刷新，自动更新。

这些场景比聊天要简单许多，仅仅涉及到监听对象的订阅、取消订阅。正如上文所提到的，使用 MQTT 实现最为经济。用社交类、直播类的思路来做，也可以实现，但略显冗余。

#### 电梯场景（假在线状态处理）

iOS 端的假在线状态有双向 ping pong 机制和 iOS 端只走 APNs 两种方案。其中，双向 ping-pong 机制原理如下：

Message 在发送后，在服务端维护一个表，一段时间内，比如15秒内没有收到 ack，就认为应用处于离线状态，先将用户踢下线，转而进行推送。此处如果出现重复推送，客户端要负责去重。将 Message 消息当作服务端发送的 Ping 消息，App 的 ack 作为 pong，如图7所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581846a6517ce.jpg" alt="图7 双向ping-pong机制原理" title="图7 双向ping-pong机制原理" />

图7 双向 ping-pong 机制原理

使用 APNs 来进行聊天的优缺点具体如下：

- 优点：解决了 iOS 端假在线的问题。

- 缺点：（APNs 的缺点）无法保证消息的及时性；让服务端负载过重。

APNs 不保证消息的到达率，消息会被折叠，你可能见过图8的推送消息。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581846b1cbc60.jpg" alt="图8 消息推送折叠" title="图8 消息推送折叠" />

图8 消息推送折叠

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581846c4678e2.jpg" alt="图9 数据传输格式占比" title="图9 数据传输格式占比" />

图9 数据传输格式占比

这中间发生了什么？

当 APNs 向你发送了4条推送，但是你的设备网络状况不好，在 APNs 那里下线了。这时 APNs 到你的手机链路上有4条任务堆积，APNs 的处理方式是，只保留最后一条消息推送给你，然后告知推送数。那么其他三条消息呢？会被 APNs 丢弃。

有一些 App 的 IM 功能没有维持长连接，是完全通过推送来实现的。通常情况下，这些 App 也已经考虑到这种丢推送的情况，它们的做法都是，每次收到推送后，向自己的服务器查询当前用户的未读消息。但是 APNs 也同样无法保证这四条推送能至少有一条到达你的 App。很遗憾地告诉这些 App，此次更新对你们所遭受的这些坑，没有改善。

为什么这么设计？APNs 的存储-转发能力太弱，大量的消息存储和转发将消耗 Apple 服务器资源，可能是出于存储成本考虑，也可能是因为 Apple 转发能力太弱。总之结果就是 APNs 从来不保证消息的达到率，并且设备上线之后也不会向服务器上传信息。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581846df30e20.jpg" alt="图10 数据传输格式投票结果" title="图10 数据传输格式投票结果" />

图10 数据传输格式投票结果

现在我们可以保证消息一定能推送到 APNs 那里，但是 APNs 不保证帮我们把消息投递给用户。即使搭配了这样的策略：每次收到推送就拉历史记录的消息，一旦消息被 APNs 丢弃，能会在几天之后收到新推送后才被查询到。

让服务端负载过重：APNs 的实现原理决定了必须每次收到消息后，拉取历史消息。

结论：如果面向的目标用户对消息的及时性并不敏感，可以采用这种方案，比如社交场景。

#### 技术实现细节

**基于 WebSocket 的 IM 系统**

WebSocket 是 HTML5 开始提供的一种浏览器与服务器间进行全双工通讯的网络技术。WebSocket 通信协议于2011年被 IETF 定为标准 RFC 6455，WebSocket API 被 W3C 定为标准。

在 WebSocket API 中，浏览器和服务器只需要做一个握手的动作，然后浏览器和服务器之间就形成了一条快速通道，由此两者间就直接可以数据互相传送。

只从 RFC 发布的时间看来，WebSocket 要晚些，HTTP 1.1是1999年，WebSocket 则是12年之后了。WebSocket 协议的开篇就说，本协议的目的是为了解决基于浏览器的程序需要拉取资源时必须发起多个 HTTP 请求和长时间的轮询问题而创建。可以达到支持 iOS、Android、Web 三端同步的特性。

### 针对移动网络特点的性能调优

**极简协议，传输协议 Protobuf**

使用 Protocol Buffer 减少 Payload，微信也同样使用定制后的 Protobuf 协议。

携程是采用新的 Protocol Buffer 数据格式+Gzip 压缩后的 Payload 大小降低了15%-45%。数据序列化耗时下降了80%-90%。

采用高效安全的私有协议，支持长连接的复用，稳定省电省流量：

1.  高效，提高网络请求成功率，消息体越大，失败几率越大。

2. 流量消耗极少，省流量。一条消息数据用 Protobuf 序列化后的大小是 JSON 的1/10、XML 格式的1/20、是二进制序列化的1/10。同 XML 相比， Protobuf 性能优势明显。它以高效的二进制方式存储，比 XML 小3到10倍，快20到100倍。

3.  省电

4. 高效心跳包，同时心跳包协议对 IM 的电量和流量影响很大，对心跳包协议上进行了极简设计：仅1 Byte。

5. 易于使用，开发人员通过按照一定的语法定义结构化的消息格式，然后送给命令行工具，工具将自动生成相关的类，可以支持 Java、C++、Python、Objective-C 等语言环境。通过将这些类包含在项目中，可以很轻松地调用相关方法来完成业务消息的序列化与反序列化工作。原生支持 C++、Java、Python、Objective-C 等多达10余种语言。2015年08月27日，Protocol Buffers v3.0.0-beta-1 中发布了 Objective-C（Alpha）版本，而在此后的两个月前，Protocol Buffers v3.0.0 正式版发布，正式支持 Objective-C。

6. 可靠，微信和手机 QQ 这样的主流IM应用也早已在使用它。

如何测试：

对数据分别操作100，1000，10000和100000进行了测试，如图11所示，纵坐标是完成时间，单位是毫秒。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581846f1f1268.jpg" alt="图11 数据测试结果（来源：beyondbit博客）" title="图11 数据测试结果（来源：beyondbit博客）" />

图11 数据测试结果（来源：beyondbit 博客）

图12为 thrift-protobuf-compare（wiki 地址 https://github.com/eishay/jvm-serializers/wiki），测试项为 Total Time，也就是指一个对象操作的整个时间，包括创建对象，将对象序列化为内存中的字节序列，然后再反序列化的整个过程。从测试结果可以看到 Protobuf 的成绩很好。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58184700d1f3e.jpg" alt="图12 thrift-protobuf-compare" title="图12 thrift-protobuf-compare" />

图12 thrift-protobuf-compare

缺点：不能表示复杂的数据结构，但 IM 服务已经足够使用。

另外，可能会造成 App 的包体积增大，通过 Google 提供的脚本生成的 Model，会非常“庞大”，Model 一多，包体积也就会跟着变大。但在我们 SDK 中只使用了一个 Model，所以这个问题并不明显。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581847276cdfc.jpg" alt="表2 TCP保活（TCP Keep Alive机制）和心跳保活区别" title="表2 TCP保活（TCP Keep Alive机制）和心跳保活区别" />

表2 TCP 保活（TCP Keep Alive 机制）和心跳保活区别

#### 在安全上需要做哪些事情

**防止 DNS 污染**

DNS 出问题的概率其实比大家感觉要大，首先是 DNS 被劫持或者失效，2015年初业内比较知名的就有 Apple 内部 DNS 问题导致 App Store、iTunes Connect 账户无法登录；京东因为 CDN 域名付费问题导致服务停摆。

另一个常见问题就是 DNS 解析慢或者失败，例如中国运营商网络的 DNS 就很慢，一次 DNS 查询甚至都能赶上一次连接的耗时，尤其 2G 网络情况下，DNS 解析失败是很常见的。因此如果直接使用 DNS，对于首次网络服务请求耗时和整体服务成功率都有非常大的影响。这一部分文章较长，单独成篇，链接http://geek.csdn.net/news/detail/111130。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/5818471711ba3.jpg" alt="图13 保活，究竟保的是谁？" title="图13 保活，究竟保的是谁？" />

图13 保活，究竟保的是谁？

**账户安全**

IM 服务账号密码一旦泄露，危害更加严峻，尤其是对于消息可以漫游的类型。在此，分享下我们的团队是如何保障账号安全的。

1.帐号安全

无侵入的权限控制：与用户的用户帐号体系完全隔离，只需要提供一个 ID 就可以通信，接入方可以对该 ID 进行 MD5 加密后再进行传输和存储，保证开发者用户数据的私密性及安全。

2.签名机制

对关键操作，支持第三方服务器鉴权，保护你的信息安全。

3.单点登录

**重连机制**

- 精简心跳包，保证一个心跳包大小在10字节之内；

- 减少心跳次数：心跳包只在空闲时发送；从收到的最后一个指令包进行心跳包周期计时而不是固定时间。

- 重连冷却2的指数级增长2、4、8，消息往来也算作心跳。类似于 iPhone 密码的错误机制，冷却单位是5分钟，10次输错，清除数据。

这样灵活的策略也同样决定了，只能在 App 层进行心跳 ping。

这里有必要提一下重连机制的必要性，我们知道 TCP 也有保活机制，但与我们在这里讨论的“心跳保活”机制是有区别的。

比如：考虑一种情况，某台服务器因为某些原因导致负载超高，CPU  100%，无法响应任何业务请求，但是使用 TCP 探针则仍旧能够确定连接状态，这就是典型的连接活着但业务提供方已死的状态。对客户端而言，这时的最好选择就是断线后重新连接其他服务器，而不是一直认为当前服务器是可用状态，总是向当前服务器发送些必然会失败的请求。

#### 使用 HTTP/2减少不必要的网络连接

大多数移动网络（3G）并不允许一个给定 IP 地址超过两个并发 HTTP 请求，即当你有两个针对同一个地址的连接时，再发起的第三个连接总会超时。而 2G 网络下这个限定是1个，同一时间发起过多的网络请求不仅不会起到加速的效果，反而有副作用。

另一方面，由于网络连接很是费时，保持和共享某一条连接就是一个不错的选择，比如短时间内多次的 HTTP 请求，使用 HTTP/2就可以达到这样的目的。

HTTP/2是 HTTP 协议发布后的首个更新，于2015年2月17日被批准。它采用了一系列优化技术来整体提升 HTTP 协议的传输性能，如异步连接复用、头压缩等等，可谓是当前互联网应用开发中，网络层次架构优化的首选方案之一。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581847276cdfc.jpg" alt="表2 TCP保活（TCP Keep Alive机制）和心跳保活区别" title="表2 TCP保活（TCP Keep Alive机制）和心跳保活区别" />

表2 TCP 保活（TCP Keep Alive 机制）和心跳保活区别

HTTP/2也以高复用著称，而且如果我们要使用 HTTP/2，那么在网络库的选择上必然要使用 NSURLSession，所以 AFN2.x 也需要升级到 AFN3.x。

**设置合理的超时时间**

过短的超时容易导致连接超时的事情频频发生，甚至一直无法连接，而过长的超时则会带来等待时间过长、体验差的问题。就目前来看，对于普通的 TCP 连接，30秒是个不错的值，而 HTTP 请求可以按照重要性和当前网络情况动态调整，尽量将超时控制在一个合理的数值内，以提高单位时间内网络的利用率。

**图片视频等文件上传**

图片格式优化在业界已有成熟的方案，例如 Facebook 使用的 WebP 图片格式，已经被国内众多 App 使用。分片上传、断点续传、秒传技术。

- 文件分块上传：因为移动网络丢包严重，将文件分块上传可以使得一个分组包含合理数量的 TCP 包，使得重试概率下降，重试代价变小，更容易上传到服务器；

- 提供文件秒传的方式：服务器根据 MD5、SHA 进行文件去重；

- 支持断点续传；

- 上传失败，合理的重连，比如3次。

**使用缓存**

微信不用考虑消息同步问题，因为它是不存储历史记录的，卸载重装消息记录就会丢失。所以我们可以采用一个类似 E-Tag、Last-Modified 的本地消息缓存校验机制。具体做法是，当我们想加载最近10条聊天记录时，先将本地缓存的最近10条做一个 hash 值，将其发送给服务端，服务端将最近十条做 hash，如果一致就返回304。最理想的情况是服务端一直返回304，一直加载本地记录，这样做的好处是消息同步、节省流量。