## QQ 会员活动运营平台的架构设计实践

文/徐汉彬

QQ 会员活动运营平台（AMS），是 QQ 会员增值运营业务的重要载体之一，承担海量活动运营的 Web 系统。在过去三年多的时间里，AMS 日请求量从200-500万的阶段，一直增长到日请求3-5亿，最高 CGI 日请求达到8亿。在这个过程中，AMS 在架构方面发生了大幅度的调整和变迁，我们走过了一段非常难忘的技术历程。本文主要分享 QQ 会员活动运营平台的架构设计实践。

### 海量活动运营的挑战和我们的应对思路

当我们说起“活动”，很多人的第一反应会觉得这是一个并不会有很多技术难度的东西。通常来说，如果我们只做一两个活动，的确没有太多技术难度，但如果将这个量级提升到做1000，甚至更多的时候，就另当别论了。

#### 活动运营业务的挑战和难题
##### **腾讯 SNG 增值业务面临海量活动运营开发的挑战**
腾讯的增值产品部在 QQ 会员体系、游戏运营、个性化等各个业务上都需要持续高强度的运营性活动来促进用户的拉新、活跃和留存，这里本身已经产生了非常多的运营需求。而且，自2014年开始，随着移动互联网迈向成熟阶段，手 Q 平台上的手游运营需求大爆发，一个月需要上线的活动出现数倍增长。
##### **活动开发的复杂性**
一个典型的大型活动通常有数千万用户参与，因此，对性能要求比较高，如果再涉及“秒杀”或者“抢购”类型的高并发功能，对于基础支撑系统是一个高难度的挑战。活动功能众多，包括礼包、抽奖、分享、邀请、兑换、排行、支付等，这些不同的参与和表现形式，也会涉及更多的后端接口通信和联调。例如，我们的游戏运营业务涉及上百款游戏，而不同的游戏对应不同的服务接口，仅游戏相关的通信接口，就涉及上千个。还有一个非常重要的问题，就是活动运营的安全性和可靠性。
##### **活动运营开发人力难题**
传统手工开发模式，普通活动的开发周期至少要1周时间，而典型大型活动更是需要1-2周，开发和测试工作量繁重。并且，很多活动是在指定节假日推广，通常有严格上线时间要求。在紧迫并且快速增长的运营需求面前，人力非常有限。 目前，我们每月上线400+活动项目，全年超过4000个。 

#### 活动本质和我们的方法论
通过对不同业务活动模式的分析和抽象，我们发现事实上绝大部分活动都可以用一组“条件”和“动作”的方式进行抽象和封装，进而形成通用的“条件”（Rule）和“动作”（Operation）活动组件，不同条件和动作的组合使用，变成活动逻辑的实现。然后，我们希望通过平台化和框架驱动开发的方式，将这些组件统一封装。同时，在框架和平台层面，为活动组件的运行提供高可靠、高性能、具备过载保护和水平扩展能力的框架支撑环境。

活动组件只需要封装自身业务逻辑，核心功能框架自动支持，从而实现活动运营开发的彻底自动化，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a162eccab2.png" alt="图1  活动本质和解决方法论" title="图1  活动本质和解决方法论" />

图1  活动本质和解决方法论

AMS 所需要承担的任务，就是实现这个规划。主要是解决三个方面的问题：
1. 建设高效活动开发模式（运营开发自动化）。 
2. 搭建高可靠性和高可用性的运营支撑平台。 
3. 保证活动运营业务的安全。

### 构建高效活动运营开发模式

2012年初，也就是在 AMS 产生之前的活动开发模式，相对比较随意，并没有一套严格和完整的框架支持，组件的复用程度不够高。因此，我们开发一个活动，经常需要耗时1周多。当时，开发活动的其中一个特点就是“各自为政”，每个运营开发同学，各自产生了一批前端和后端组件，CGI 层也产生了很多不同规则的入口。这些各自实现的组件，结构较凌乱，不成体系，维护起来也困难。最重要的是，这样的组件对于活动开发来说，使用复杂，复用率低，以至于开发效率也较低。

当时，活动运营需求也出现了一定程度堆积，很多需求没有人力支持，产品同学也觉得上线活动比较慢，如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a1680c3585.png" alt="图2  前期AMS“各自为政”的开发结构图" title="图2  前期AMS“各自为政”的开发结构图" />

图2  前期 AMS“各自为政”的开发结构图

#### 系统架构分层和统一
基于这个问题，我们当时想到的第一个解决方案，就是整合前端和后端组件，重新搭建一个结构清晰和统一的系统。将这个系统的接口分层、复用、简化的原则，逐步构建一个完整的体系。而且，从我们开发的角度来说，最重要的目的，是为减少活动开发的工作量，解放开发人员，提升研发效率，如图3所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a16b302145.png" alt="图3  整合后的AMS开发结构图" title="图3  整合后的AMS开发结构图" />

图3  整合后的 AMS 开发结构图

我们的前端组件通过一个叫 Zero 的框架统一整合，前端每一个功能以组件的形式出现，统一维护和复用。CGI 层则进行了代码重构，实行框架驱动式开发，将每一个业务逻辑功能，收归到一个唯一的入口和统一的体系中。核心功能框架自动支持，已有活动功能组件可直接配置使用。如果没有新的功能接入，运营开发只需要配置一份简单的参数，就可以完成后端功能逻辑，不再需要写代码。对于基础支撑服务，则以平台化的模式进行管理，做统一接入和维护。

做完系统结构的调整后，我们终于实现通过一份活动配置，来控制前端和后端的组件组合。每一个条件、发货等动作，都可以随意动态组合，参与条件通过“与”、“或”、“非”等组合方式，选择对应的动作，实现活动功能逻辑。

从那时开始，活动开发变得简单了不少，需要写的代码大幅度减少，基本变成“填写参数”的工作。一个活动项目的代码从之前的1000-2000行，变成了不到100行。如图4所示，本来需要写不少逻辑代码的领取礼包，在前端只变成了一行参数。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a16d83a1af.png" alt="图4  活动开发变简单的实现效果图" title="图4  活动开发变简单的实现效果图" />

图4  活动开发变简单的实现效果图

清晰的结构提升了系统可维护性，更为重要的是，活动开发效率也得到了极大的提升。

在开发人力不变的情况下，我们活动开发的效率实现了大幅提升，产品需求积压的情况，得到有效缓解。

#### 高可视化开发模式（自动化运营）
然而，到了2014年，随着“移动互联网”的快速发展和逐步成熟，我们也迎来了“手游大爆发”时代。因为手游的开发周期更快，几乎每个月都有很多款新手游上线，很快手游活动运营的需求出现了爆发式增长。AMS 承担的活动需求，迅速从每个月上线60多个上升到200个的量级，在此背景下，开发人力再次捉襟见肘，需求的积压问题进一步加剧。

既然说到开发人力，就必须介绍一下我们当前的活动项目模式。腾讯是一家成熟的互联网公司，研发流程的每一个环节（设计、重构、开发、体验/测试、发布），都由不同独立角色完成。一个普通的移动端活动项目耗时，按照最快速、最理想的模式计算：设计1天，重构1天，开发2天，体验/测试1天，也至少需要5个工作日，也就是研发周期至少1周。理想是美好的，现实总是残酷的，在实际项目实施过程中，因为各种资源协调和外部因素影响，通常无法达到如此完美的配合，一个普通活动的研发周期，往往都超过1周。

忽然新增100多个需求，无论对于任何团队来说，都是一个巨大的压力。

于是，我们不得不采用另外一种思路，来看待活动运营，是否可以尝试不投入开发人力？我们称之为“自动化运营“， 自动化的本质，就是构建足够强大的平台和工具支撑，让运营同学自己完成活动开发。

前面，我们提到，开发普通活动时，每一个功能点已经变成了一份简单的配置，而活动开发的工作，就是将这个配置的活动参数填入到页面按钮上。如果，我们实现一个可视化工具，将这个填写配置的工作，变成拖拽按钮的功能，这样就可以彻底告别“写代码”的工作，如图5所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a171510461.png" alt="图5  可视化拖拽的活动模板系统" title="图5  可视化拖拽的活动模板系统" />

图5  可视化拖拽的活动模板系统

最终的结果，是我们制做一个可视化拖拽的活动模板系统。运营同学只需要经过适当培训，就能学会如何使用。自动化运营给我们带来了研发流程级别的优化，在活动研发流程中，减少了重构、开发和测试的流程，使得活动项目研发周期大幅度缩短，活动项目研发效率出现质的飞跃。手游运营需求的积压问题，得到根本和彻底的解决。 

高效活动开发模式构建完成，也促使我们的 AMS 平台业务规模快速增长。一个月上线的活动项目数，在2015年10月超过400个，而其中有80%以上属于运营同学“开发”的模板活动。

### 可靠性与性能支撑建设

通过构建高效的活动开发模式，促使我们 AMS 运营平台的业务规模和流量规模，都在过去的三年多时间里，出现了100倍的增长，同时在线的活动超过1000个。与此同时，AMS 平台的可靠性和稳定性，也成为至关重要的指标之一，平台如果出问题，影响面变得很广。

AMS 平台的架构分为四个层级，分别为：入口层、业务逻辑层、服务层、存储层（全部是腾讯自研的NoSQL数据），还有一个离线服务和监控系统，如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a174c700d6.png" alt="图6  AMS平台的架构" title="图6  AMS平台的架构" />

图6  AMS 平台的架构

#### 可靠性
活动运营业务，对平台的可靠性非常敏感，因为这里涉及到很多高价值礼包的发放，部分还涉及支付环节，稳定压倒一切。

在保证可用性方面，我们做了几个方面的工作：
1. 所有服务与存储，无状态路由。这样做的目的，主要是为了避免单点风险，就是避免某个服务节点挂了，整个服务就瘫痪了。实际上，即使像一些具有主备性质（主机器挂了，支持切换到备份机器）的接入服务，也是不够可靠的，毕竟只有2台，它们都挂了的情况，还是可能发生。后端的服务，通常都以一组机器的形式提供，彼此之间没有状态关系，支撑随机分配请求。
2. 支持平行扩容。遇到大流量场景，支持加机器扩容。
3. 自动剔除异常机器。在我们的路由服务，发现某个服务的机器异常的时候，支持自动剔除，等它恢复之后，再重新加回来。
4. 监控告警。实时统计成功率，当低于某个阀值，告警通知服务负责人。

在告警监控方面，AMS 平台的建设更为严格，我们力求多渠道告警（RTX、微信、邮件、短信），多维度监控（L5、模块间调用、自动化测试用例、AMS 业务监控维度等）。即使某些监控维度失效，我们同样可以第一时间发现问题。当然，我们也会控制告警的周期和算法，做到尽量减少骚扰，同时，又能真正发现系统问题。

可靠性另外的一个挑战，就是过载保护。不管系统拥有多少机器，在某些特殊场景下，终究有过载的风险，例如“秒杀”和“定时开启”之类的推广面前。AMS 当前同时在线的活动超过1000，已经太多了，这些活动中，偶尔总会有大流量推广，并且业务方甚至根本没有周知到我们。无论在何种场景下，我们必须做到 AMS 平台本身不能“雪崩”，如果集群挂掉，就影响全量用户，而做过载保护只是抛弃掉了部分用户请求，大部分用户还是能够获得正常的服务。

在过载保护方面，我们采取了一些并不复杂的措施：
1. 平台入口过载保护。经过业务特性和机器运营经验，设置 Apache 最大服务进程/线程数，确保在这一个数目下，不会出现宕机或者服务挂掉。 
2. 后端众多发货接口的保护。AMS 平台的背后涉及数以百计的后端服务接口，他们性能参差不齐。通过 AMS 平台内部限流，保护弱小的后台接口。因为，如果将它们压垮了，就会造成某类型的接口完全不可用。
3. 服务降级。旁路非关键路径，例如数据上报服务。 
4. 核心服务独立部署，物理隔离。鸡蛋不能放在同一个篮子里，物理隔离的目的，就是避免业务之间相互影响，保护其他服务正常工作。

#### 秒杀场景的业务保护
秒杀在活动运营中，是比较常见的一种参与形式，它带来的挑战除了流量冲击，还有高并发下的业务逻辑安全。这个时候，我们必须引入适当的锁机制来规避。它和线程安全是同一类型的问题。

首先是用户的 Session 锁，也就是说，同一个子活动功能中，同一个用户，在前一次发货请求结束之前，禁止第二个请求。之所以要这样做，是因为，如果同一个用户发起两个并发请求，在一个临界时间内，可能导致礼包多发。还有一个是基于多个用户的秒杀保护锁，场景类似，Session 锁，只是变为多个并发用户，请求同一个礼包，同样在判断礼包余量数目的临界时间里，有可能产生“超发”（礼包多发了）。

问题很明显，采用锁当然就可以解决，但采用何种锁机制，又是一个值得思考的问题。因为业务场景不同，选择的解决方案自然不同。我们从三个不同的思路，来讨论秒杀的实现机制。

1. 队列服务。直接将秒杀的请求放入队列中，逐个执行队列里的请求，就像强行将多线程变成单线程。然而，在高并发的场景下，请求非常多，很可能一瞬间将队列内存“撑爆”，然后系统又陷入到了异常状态。或者设计一个极大的内存队列，也是一种方案，但是，系统处理完一个队列内请求的速度，通常无法和疯狂涌入队列中的请求数相比。也就是说，队列内的请求会越积累越多，并且排在队列后面的用户请求，要等很久才能获得“响应”，无法做到实时反馈用户请求。
2. 悲观锁思路。悲观锁具有强烈的独占和排他特性，用户请求进来以后，需要尝试去获取一个锁资源，获得锁资源的请求可执行发货，获取失败的则尝试等待下一次抢夺。但是，在高并发场景下，这样的抢夺请求非常多，并且不断增加，导致这种请求积压在服务器。绝大部分请求一直无法抢夺成功，一直在等待（类似线程里的“饿死”）。用户也同样无法获得实时响应和反馈。
3. 乐观锁思路。相对于“悲观锁”采用更为宽松的加锁机制，采用带版本号 （Version）更新。实现就是，这个数据所有请求都有资格去修改（执行发货流程），但会获得一个该数据的版本号，只有版本号符合的才能更新成功（版本不相符，也就是意味着已经被某个请求修改成功了），其他用户请求立刻返回抢购失败，用户也能获得实时反馈。这样的好处，是既可以实现锁的机制，又不会导致用户请求积压。

我们采用的是乐观锁实现方式，因为一个发货流程通常耗时超过 100 ms，在高并发下，都容易产生请求积压，导致无法做到实时反馈。我们的实现，确保不管用户是否请求秒杀成功，都能在 500 ms 内获得实时反馈。并且，我们将这个实现广泛使用到各个秒杀和抢购活动中，曾经支撑过 5w/s 的秒杀活动，表现非常平稳和安全。

### 业务安全体系建设
随着业务规模的增长，AMS 平台每天发出的发货操作也越来越多。在非节假日每天达到5000多万，高峰时超过2亿。

#### 传统安全打击维度和恶意用户
成熟的互联网公司通常都有自己的安全团队，一般通过数据建模的方式，搭建出一个恶意用户黑名单数据库，然后持续维护这些恶意账号和 IP 等信息，更新数据。然后，我们这个服务接入到里面去。恶意工作室手持大量账号和 IP，而我们通过这个恶意数据库，将它们拦截。

但是，数据建模的算法不管如何精细，为了防止误杀真实用户，总会存在打击率的问题，它们通常无法拦截下全部恶意请求，总会有少数的漏网之鱼。

而我们所思考的，就是在这个基础上，结合业务增加新的安全保护策略。可能有很多人会想，追加参与门槛是否可以取得进一步的保护效果呢？例如，在传统安全打击策略的基础上，再加上业务限制，如将活动参与条件设置为超级会员，我们以更高的门槛来拦截恶意请求。但付费会员身份限制，也是不可靠的，超级会员的身份带给这些恶意号码更多的便利，反而可以给它们获取更多高价值礼包的机会，将获得东西兑现成金钱，然后覆盖恶意工作室的“投资成本”。

#### HTTP 和手 Q 的 SSO 通道
恶意用户参与活动方式，通常是通过脚本模拟请求，并不是真实用户的常规点击。其中，这里面有一点引起了我们的注意，就是 HTTP 协议。这是个公开、透明的传输协议，我们大家都知道如何构造一个 HTTP 请求包，模拟活动请求行为。如果不使用这个协议，而采用更为私密的通信协议，是否可以提升请求伪造的成本？

因为我们的活动，大部分都是在手 Q 上推广，用户实际上是在手 Q 的内置 Webview（内置浏览器）里参加活动，我们也有机会更好地复用终端的能力。于是，我们利用手 Q 的私有加密传输协议 SSO，来传输活动参与请求，然后禁止来自于 HTTP 协议的请求。

SSO 通道还有另外一个好处，它本质上是一个多地部署的网络通信服务，用于提高网络传输效率。类似 CDN，在各个地区部署节点，传输数据会选择地理位置最近的节点，然后开始走内网专线，将通信数据传输给指定服务，避开传输效率不高的公网。

当在高价值礼包上开启 SSO 保护之后，恶意用户的请求数量得以大幅度下降。我们对比过一组高价值礼包的活动，恶意请求从41%下降到3%。

#### 手机 IMEI 保护和限制
IMEI 是手机的唯一识别码，类似于电脑的 MAC 地址。借助手 Q 终端，我们在对用户的请求维度，增加到账号、IP、IMEI。充分利用这个新的识别维度，将它和 QQ 账号关联到一起，因为在一般情况下，一个用户只在一台手机上登录和使用 QQ。基于这个特点，我们在高价值礼包上增加了一个新的保护策略，就是 IMEI 和 QQ 关联，作为新的拦截判断维度。

也许有一些同学说，手机的 IMEI 也可以伪造，理论上是可以的，但是具有一定的门槛。实际上，并不存在某种安全策略，能够将全部恶意请求拦截，更切合实际的，是通过多种安全保护策略叠加在一起，发挥出更大的效果，将绝大部分恶意请求拦截。

经过我们实际应用发现，在一些珍贵的礼包场景下，即使在传统保护策略和 SSO 保护开启的基础上，IMEI 保护策略还能取得20%的拦截率。

#### 业务安全支撑体系建设
AMS 除了上面介绍的三种入口类型的安全策略，还建设多个维度，全方面的安全支撑能力，分为四个维度。
1. 入口安全：用户对接的 CGI 入口，过滤恶意和攻击请求，保护业务逻辑安全等。
2. 人的失误：开发和运营同学在管理端的误操作或者配置错误，人的失误不能靠人去保证，而应该建设平台级别的权限管理和智能检测，让人不容易“失误”。
3. 运营监控：多维度建设对业务状态的监控，确保快速发现问题。
4. 审计安全：将敏感的权限充分收归、管理、监控，确保可控可追溯。