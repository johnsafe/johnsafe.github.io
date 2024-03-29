## 饿了么移动 App 的架构演进

文/胡彪

随着互移动互联网时代的到来，移动技术也随即迎来新的发展高峰。如今，App 已然成为绝大多数互联网企业用来获取用户的核心渠道。与此同时，伴随着业务量的增长，愈来愈大、愈来愈多的 App 也在不断地、持续地挑战着每一个移动端研发人员的知识深度，也正是这些不断被突破的挑战，成就了今天的移动互联网时代。饿了么作为一家在 O2O 领域高速发展的公司，App 端面临着多重挑战：庞大的用户群体、高频高并发的业务、交易即时性等等。移动端的开发小伙伴在技术和业务的双重压力下，不断前进，推动着饿了么移动端的架构演进。

### MVC

我们常说，脱离业务谈架构就是纯粹的耍流氓。饿了么移动 App 的发展也是其业务发展的一面镜子。 

在饿了么业务发展的早期，移动 App 经历从无到有的阶段。为了快速上线抢占市场，传统移动 App 开发的 MVC 架构成了“短平快”思路的首选。

这种架构以层次结构简单清晰，代码容易开发而被大多数人所接受。

在 MVC 的体系架构中，Controller 层负责整个App中主要逻辑功能的实现；Model层则负责数据结构的描述以及数据持久化的功能；而 View 层作为展现层负责渲染整个 App 的 UI，分工清晰、简洁明了；并且这种系统架构在语言框架层就得到了 Apple 的支持，所以非常适用于 App 的 Startup 开发。

然而，这种架构在开发的后期会由于其超高耦合性，从而造就庞大 Controller 层，而这也是其一直被人所诟病之处。最终 MVC 都从 Model-View-Controller 走向了 Massive-View-Controller。
<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a2543bad15.png" alt="图1  MVC架构" title="图1  MVC架构" />

图1  MVC架构

### Module Decoupled、

“短平快”的 MVC 架构在早期可以满足饿了么移动 App 的快速开发迭代，但是随着代码量的不断增加，臃肿的 Controller 层也在渐露端倪；而业务上，饿了么移动 App 也从单一 App 发展为多 App 齐头并进的格局。这时候，如何降低耦合，复用已有模块成了架构的第一要务。

架构中，模块复用的第一要求便是代码的功能组件化。组件化意味着拥有独立功能的代码从系统中进行抽象并剥离，再以“插件”的形式插回原有系统中。这样剥离出来的功能组件，便可以供其他 App 进行使用，从而降低系统中模块与模块之间的耦合；也同时提高了 App 之间代码的复用。

饿了么移动对于组件有两种定义：公有组件和业务组件。公有组件指的是封装得比较好的 SDK，包括一些第三方组件和自己内部使用的组件。如 iOS 中最著名的网络 SDK AFNetworking、Android 下的 OKHttp，都是这类组件的代表。而对于业务组件，则定义为包含了一系列业务功能的整体，例如登录业务组件、注册业务组件，即为此类组件的典型代表。

对于公有组件，饿了么移动采取了版本化的管理方式，而这在 iOS 和 Android 平台上早已有比较成熟的解决方案。例如，对于 iOS 平台，CocoaPods 基本上成为了代码组件化管理的标配；在 Android 平台上，Gradle 也是非常成熟和稳健的方案。采用以上管理工具的另一个原因在于，对企业开发而言，代码也是一种商业机密。基于保密目的，支持内网搭建私有服务器成为了必需。以上的管理工具都能够很好地支持这些操作。

对于业务的组件化，我们采取了业务模块注册机制的方式来达到解耦合的目的。每个业务模块对外提供相应的业务接口，同时在系统启动的时候向 Excalibur 系统注册自己模块的 Scheme（Excalibur 是饿了么移动用来保存 Scheme 与模块之间映射的系统，同时能根据 Scheme 进行 Class 反射返回）。当其他业务模块对该业务模块有依赖时，从 Excalibur 系统中获取相关实例，并调用相应接口来实现调用，从而实现了业务模块之间的解耦目的。

而在业务组件，即业务模块的内部，则可以根据不同开发人员的偏好，来实现不同的代码架构。如现在讨论得比较火的 MVVM、MVP 等，都可以在模块内部进行而不影响整体系统架构。
这时候的架构如图2所示。
<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a25a46488a.png" alt="图2  EMC架构" title="图2  EMC架构" />

图2  EMC 架构

这种
E（Excalibur）M（Modules）C（Common）架构以高内聚、低耦合为主要的特点，以面向接口编程为出发点，降低了模块与模块之间的联系。

该架构的另外一大好处则在于解决了不同系统版本的兼容性问题。这里举 iOS 平台下的 WebView 作为例子来进行说明。Apple 从 iOS 8系统开始提供了一套更好的 Web 支持框架——WebKit，但在 iOS 7系统下却无法兼容，从而导致 Crash。使用此类架构，可以在 iOS 7系统下仍然注册使用传统的 WebView 来渲染网页，而在 iOS 8及其以上系统注册 WebKit 来作为渲染网页的内核。既避免了 Apple 严格的审核机制，又达到了动态加载的目的。

### Hybrid

移动 App 的开发有两种不同的路线，Native App 和 Web App。这两种路线的区别类似于 PC 时代开发应用程序时的 C/S 架构和 B/S 架构。

以上我们谈到的都属于典型的 Native App，即所有的程序都由本地组件渲染完成。这类 App 优点是显而易见的，渲染速度快、用户体验好；不过缺点也十分突出：出现了错误一定要等待下一次用户进行 App 更新才能够修复。

Web App 的优点恰好就是 Native App 的缺点，其页面全部采用 H5撰写并存放在服务器端。每次进行页面渲染时都从服务器请求最新的页面。一旦页面有错误服务器端进行更新便能立刻解决。不过其弊端也容易窥见：每次页面都需要请求服务器，造成渲染时等待时间过长，从而导致的用户体验不够完美，并且性能上较 Native App 慢了1-2个数量级；与此同时还会导致更多的用户流量消耗。另一个缺点则在于，Web App 在移动端上调用本地的硬件设备存在一定的不便。不过这些弊端也都有相应的解决方案，如 PhoneGap 将网页提前打包在本地以减少网络的请求时间；同时也提供一系列的插件来访问本地的硬件设备。然而，尽管如此，其渲染速度上还是会存在细微的差距。

Hybrid App 则是综合了二者优缺点的解决方案。饿了么移动对于此二类 App 的观点在于，纯粹展示性的模块会更适合使用 Web 页面来达到渲染的目的；而更多的数据操作性、动画渲染性的模块则更适合采用 Native 的方式。

基于之前的 EMC 架构，我们将部分模块重新进行了架构，如图3所示。
<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a25ed06e54.png" alt="图3  Hybrid EMC架构" title="图3  Hybrid EMC架构" />

图3  Hybrid EMC 架构

Hybrid-EMC 架构中，Web 作为一个子模块，注册加入到整个系统中，从而实现让业务上需要快速迭代的模块达到实时更新的效果。

### React-Native & Hot Patch

经过这些年的业务发展，Hybrid 提供的展示界面更新方案也逐渐地无法满足 App 更新迭代的需要。因此越来越多的动态部署的方案被提了出来，比如 iOS 下的 JSPatch、waxPatch，Android 下的 Dexpose、AndFix、ClassLoader，都是比较成熟 Hot Patch 动态部署解决方案。这些方案的思路都是通过下载远程服务器的代码来动态更新本地的代码行为。

React-Native 则属于另一种动态部署的方案，其核心原理在于通过 JavaScript 来调用本地组件进行界面的渲染。

而饿了么移动 App 发展到今天，各个 App 综合用户量已经过亿。因此一个非常小的 Bug 所带来的问题都可能会直接影响到几万人的使用。为了保证 App 的稳定和健壮，Hot Patch 方案也就成了当下最有待解决的问题。

根据80%的用户访问20%页面这一80/20原则，保证这20%访问最频繁的页面的稳定性就是保证了80%的 App 的稳定性。因此，饿了么移动对于部分访问最频繁的模块进行了 React-Native 备份。当这部分页面出现问题时，App 可以通过服务器的配置，自动切换成 React-Native 的备份页面；而与此同时开发人员开发一个小而精的 Hot Patch 来修复出现的问题。当 Hot Patch 完成修补后，再切换回 Native App 的原生功能。

这时候的架构如图4所示。
<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568a268677826.png" alt="图4 HotPatch-EMC架构" title="图4 HotPatch-EMC架构" />

图4 HotPatch-EMC 架构

HotPatch-EMC 架构主要目标在于解决移动 App 的稳定性问题。通过 React-Native 与 Native 的主备，可以减少系统 App 出错带来的失误成本。

### 结语

我们都知道，对于软件工程来说，这世上没有银弹。对于架构而言其实也非常适用。移动技术的不断发展和业务的不断变化，推动了饿了么移动 App 架构的不断演进。
架构没有真正的好坏之分，只要适用于自己的业务，就是好的架构！