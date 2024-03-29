## 小米异步消息系统实践

文 / 王晓宇

本文首先介绍小米网系统架构的发展变化，然后介绍 Notify 系统设计，最后介绍其演化与升级变迁。希望能给各位的工作带来一些启发。

为了适应业务的高速发展，小米网系统架构经历了很多次变更。在此过程中，为了给各个子系统解耦合，同时保证最终一致性原则的实现，我们建立了自己的异步消息系统—— Notify。

### 小米网架构发展

小米网的发展大致可以分为三个阶段：初创、发展和完善阶段。

#### 初创阶段
当小米推出自己的第一部手机时，为了减少渠道成本，我们开始推行电商直销的商业模式，与此同时，开始建设小米电商网站。最开始，小米的业务特点是：SKU（商品品类）单一；订单量巨大；瞬时访问量巨大。

后两点是在最初设计系统时完全没有想到，因为我们并没有预料到小米手机会如此受欢迎和供不应求。

在这个阶段，快速上线是第一目标，因为团队需要快速配合公司的手机销售计划。所以一开始小米网的架构设计比较简单，并没有考虑高并发和大数据的情况。当时系统从立项到上线仅两个多月时间，并且只由三名工程师开发完成。

从图1中可以看出，系统架构只有两个 Web 服务器与一个 DB 服务器，两台 Web 服务器互为主备，所有的业务功能集成在一个系统中。当时的架构设计仅能支持简单的电商功能，我们预测每年的手机销量能到30万就已经很好了。但是计划永远赶不上变化，很快小米电商就遇到了第一个大问题：系统耦合度很高，导致抢购活动开始时，其他业务都会受到影响。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5c839aa2e.jpg" alt="图1  小米电商初期的系统架构" title="图1  小米电商初期的系统架构" />

图1  小米电商初期的系统架构

#### 发展阶段
为了解决上面的问题，需要对小米网的架构进行修改，把各种业务系统拆成独立的子系统。这一时期小米电商的系统架构发展的特点是：
1. 业务系统的拆分，小米负责处理抢购请求的大秒系统就是在这一阶段诞生的，将抢购业务带来的系统压力完全隔离开来，确保在抢购活动时其他业务可以不受影响；
2. 系统结构 SOA（面向服务的软件结构）化，各系统之间的通信采用接口方式来实现，甚至我们开发了一套通信协议，叫做 X5协议，来规范接口的开发与调用。这一阶段小米网的架构如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5d6b1b941.jpg" alt="图2 小米电商发展阶段的系统架构" title="图2 小米电商发展阶段的系统架构" />

图2 小米电商发展阶段的系统架构

从图2可以看出，前端与后端系统完全独立出来，当前端进行抢购活动时，后端的客服、售后、物流三大服务系统不会受到任何影响。图2只是一小部分较为主要的系统，还有很多业务没有列举出来。

这种架构可以确保系统之间不会受到影响，但是接口的稳定性就成了一个至关重要的问题。这种系统架构几乎所有的接口都是同步接口，意思就是说一个业务调用这些接口如果不成功，业务也无法成功。如果出现网络问题或者接口 Bug，就会导致大量业务失败。但事实上，并非所有的接口都必须做成同步，比如订单系统把订单信息发给仓储系统生产，就对同步性的要求不是很高，可以考虑使用异步方式来解决。为了解决这个问题，小米网架构下一步的演化就是建立一个异步消息队列系统。

#### 完善阶段
经过异步消息系统的引入，系统架构最终发生了变化，如图3所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5dbfa883f.jpg" alt="图3 小米网的系统架构" title="图3 小米网的系统架构" />

图3 小米网的系统架构

可以看出，异步消息队列成了一个中心节点，几乎所有的子系统都与它有单向或者双向的交互。当业务需要调用一个接口时，如果对实时性的要求不是特别高，就可以把消息发到异步消息队列系统，然后就可以完成这项业务，而之后的消息投递过程完全交给消息队列系统来实现。经过这次调整，小米网基本实现了主要业务的异步化，大大增加了系统的容错性，因为所有投递不成功的消息都可以保存在消息队列系统中等待下次投递。

### Notify 消息系统的设计

基于对上面业务变化的分析，小米网内部开始计划建立自己的异步消息系统。我们对比了市面的几款 MQ 软件，最后决定以 Redis 队列为基础，开发自己的异步消息队列系统，取名叫做 Notify 异步消息系统。

Notify 的设计需要解决以下几个问题：完善阶段如何接收消息；如何存储消息；如何投递消息；对消息的统计与监控。
 
我们采用接口的方式来接收业务系统的消息，采用 MySQL 来存储消息，在消息发送时使用 Redis 队列来存储。

为了实现以上主要功能，为 Notify 系统设计了以下数据结构。下面为五个最主要的数据表，以及重要的字段：
1. biz - 业务（生产者）；
2. receive - 接收者（消费者）；
3. biz_receive - 订阅关系；
4. biz_msg - 业务消息；
5. receive_msg - 投递消息。

这里需要提一下 Notify 系统的消息分裂机制。考虑到有可能在一项业务执行过程中需要把消息发给多个接口，Notify 消息在设计的时候引入了一个消息分裂的概念。

图4分别列举了消息分裂的三种情况。首先第一种，在没有消息队列时直接调用接口的情况，一个业务执行时如果要将一个消息传递给不同的系统，就需要调用不同的接口，并且这些接口还必须都返回成功，才能算这个业务执行成功。第二种情况是引入消息队列来处理这个问题，如不进行分裂，S1需要把同一个消息塞到不同的消息队列里去。也就是要多次将消息发送给 Notify 系统。这样设计虽然可以确保业务执行成功，但却不具备扩展性。假设我们新建立一个系统，也需要同样的消息，那么就不得不回过头来修改代码，关闭一个系统也是同理。所以第三种情况中我们设立了一个消息分裂与订阅的机制，业务执行时只需要把消息投递到 Notify 系统一次，而其他系统如果需要这个消息，就可以在我们的 Notify 系统中设置订阅关系，同样的消息就被复制成多个副本，然后被塞到多个不同的消息队列来投递。这样做既可以进一步提高业务执行的成功率，又使得业务具备可扩展性与可配置性。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5e1f1d93f.png" alt="图4  Notify消息设计中的三种消息分裂情况" title="图4  Notify消息设计中的三种消息分裂情况" />

图4  Notify消息设计中的三种消息分裂情况

基于上面的一系列设计思想，最终形成了图5中的结构，即 Notify 系统的最初设计图。我们通过 Api.notify 接口，来收集业务系统的消息，并存放在 DB 中。发送消息时，设立一个 Maker 组件，采用多进程执行方式，对每一个订阅关系开启一个进程，把消息复制一个副本并放到对应的 RedisMQ 中。MQ 的名称就以 biz-receive 的对应 id 组合而成，方便查找。然后，我们设立了一个 Sender 组件，Sender 主要完成两样工作：一是把消息发送给对应的业务系统；二是把消息放到 Marker Queue 中，来回写消息的状态。如果消息发送成功了，就把状态回写成已投递，如果发送失败，就把消息状态重新回写成待处理，以便下一个周期再次投递。然后我们又设立了一个 Marker 程序，来异步读取 Marker Queue 里的消息来回写状态。这样就完成了一个投递周期，整个 Notify 系统就是通过这种方式源源不断地将消息投递给各业务系统。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5e692f82f.png" alt="图5  Notify的架构设计" title="图5  Notify的架构设计" />

图5  Notify的架构设计

除了上面的主要结构外，在实现 Notify 时，还引入了以下几个特性：
1. 消息分裂，如上文介绍过的一样。
2. 冷库备份功能。随着业务的扩展，DB 中的消息数也增长的很快，如果不对 DB 中的消息做备份，会影响 Notify 本身的性能，以及统计功能的可用性。对于已经投递成功的消息来说，大部分情况下不会被用到，所以需要定期对消息做迁移冷库的操作。
3. 为了保证发送消息的时效性，对 Maker 与 Sender 进行了多进程编程，每个进程负责一个订阅关系的处理，可以独占一个 MQ 的控制权，我们通过这种方式来提高消息发送的时效性，确保关键业务消息不会被阻塞。
4. 消息重发功能。如果消息发送失败，会被 Marker 重新标记成待处理状态，以进入下一次的投递周期，同时消息的发次数会加1。每次投递都间隔一定的时间，当投递次数超过一个阈值时，就不再投递了。因为这个时候可能是由于业务系统的接口出了什么问题，再尝试投递没有任何意义，还会造成网络流量的浪费，影响其他系统的业务，所以停止继续投递。待接口问题修复后，我们再手动批量重推消息。
5. 采用异步方式，在投递周期中，也用到了异步思想，这样做也是为了加快消息投递的时效。
6. 消息可查询。Notify 为小米网的其他工程师提供了一个可以查询消息体及返回的地方，方便我们工程师调试系统，定位 Bug。

### Notify 消息系统的升级变迁

第一版 Notify 消息系统设计的时候还存在着很多不足与考虑不周之处。随着业务的发展，逐渐暴露出来。小米网每年有两大促销活动，一个是天猫的双十一，另外一个是小米自己的米粉节。这两个节日，小米网的订单量呈爆发式增长，而且每年的订单量峰值都会有所增加。这对业务系统，尤其是作为中心节点的 Notify 系统来说，是一次巨大的冲击。所以 Notify 消息系统在最初设计的基础上经过了很多修改，并且于去年完成了一次大重构，才达到了现在的处理能力，以目前的性能来看，小米网可以轻松经受住米粉节或双十一的订单量。

目前 Notify 系统主要的不足有以下几点。业务直接通过接口方式投递消息，如果网络出现不可靠，直接投递还是会有引起业务失败的风险；系统全部使用 PHP 实现：做为一种脚本语言，PHP 还有很多不足，比如无法进行高效的运算；采用单一 MySQL 实例：在数据量过大时会影响性能。

针对上面这些不足，我们对 Notify 进行了一些重构。首先，对于接收消息的功能，我们改为采用 Agent 代理的方式，来收集消息，如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5f2127d42.png" alt="图6  Notify采用Agent代理的方式 " title="图6  Notify采用Agent代理的方式 " />

图6  Notify 采用 Agent 代理的方式

业务系统由原来的直接将消息发送给 Notify，改为将消息存在本地数据库，然后由常驻内存的 Agent 代理来收集消息，并将消息发送给 Notify 系统。如此一来，大大增加了业务的成功率，因为 DB 操作的可靠性远大于网络操作。

对于 PHP 问题，我们则是采用 Golang 将关键模块重构。经过这次重构，主要模块的性能都有了很大提升。与老版本对比，Api.Notify 系统每台 Web 服务器接收消息的能力提升了21倍，Marker 服务处理能力提升了4倍，Sender 服务处理能力提升了4倍。最终投递周期中的各环节性能达到了图7中所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b5f582360a.png" alt="图7" title="图7" />

图7

针对单一 MySQL 实例问题，我们引入了 MyCAT，MyCAT 是阿里开发并维护的一款开源数据库中间件，实现了 MySQL 的分布式存储，提升了数据库性能，并且 MySQL 实例数量动态可扩展，最重要的是，MyCAT 可运行 MySQL 语句，因此与 MySQL 几乎无缝切换，开发成本小。

### 总结

本文主要介绍了小米网在实现异步消息队列系统时所进行的实践与探索，介绍了异步消息系统的设计经验，希望也能为其他公司的同行提供经验。