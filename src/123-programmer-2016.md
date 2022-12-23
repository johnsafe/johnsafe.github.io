## 从 iOS 视角解密 React Native 中的线程

文/彭飞

>React Native（后文简称 RN）自推出至今，已在国内不少公司得到了推广应用，前景颇为看好。而当前市面上对RN源代码级别的研究文章较少，对理解以及应用 RN 上带来诸多不便。线程管理是 RN 的一个基础内容，理清它对了解 RN 中的组件设计、事件交互、复杂任务处理有很大的帮助。由此，本文将基于 iOS 端的源代码介绍 RN 中线程管理的相关内容。

在 iOS 开发中，一谈到线程管理，肯定离不开 GCD（Grand Central Dispatch）与 NSOperation/NSOperationQueue 技术选型上的争论。关于这两者普遍的观点为：GCD 较轻量，使用起来较灵活，但在任务依赖、并发数控制、任务状态控制（线程取消/挂起/恢复/监控）等方面存在先天不足；NSOperation/NSOperationQueue 基于 GCD 做的封装，使用较重，在某些情景下性能不如 GCD，但在并发环境下复杂任务处理能很好地满足一些特性，业务扩展性较好。

但是 RN 在线程管理是如何选用 GCD 和 NSOperation 的？带着此问题，一起从组件中的线程、JSBundle 加载中的线程以及图片组件中的线程三个方面，逐步看看其中的线程管理细节。

### 组件中的线程

#### 组件中的线程交互

RN的本质是利用 JS 来调用 Native 端的组件，从而实现相应的功能。由于 RN 的 JS 端只具备单线程操作的能力，而 Native 端支持多线程操作，所以如何将 JS 的单线程与 Native 端的多线程结合起来，是 JS 与 Native 端交互的一个重要问题。图1，直观展示了 RN 是如何处理的。

先从 JS 端看起，如图1所示，JS 调用 Native 的逻辑在 MessageQueue.js 的_nativeCall 方法中。在最小调用间隔（MIN_TIME_BETWEEN_FLUSHES_MS=5ms）内，JS 端会将调用信息存储在_queue 数组中，通过 global. nativeFlushQueueImmediate 方法来调用 Native 端的功能。global.nativeFlushQueueImmediate 方法，在 iOS 端映射的是一个全局的 Block，如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dcb89c8f5.png" alt="图1  JS调用Native端" title="图1  JS调用Native端" />

图1  JS 调用 Native 端

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dcc6727b9.png" alt="图2  global.nativeFlushQueueImmediate方法映射" title="图2  global.nativeFlushQueueImmediate方法映射" />

图2  global.nativeFlushQueueImmediate 方法映射

nativeFlushQueueImmediate 在这里只是做了一个中转，功能的实现是通过调用 RCTBatchedBridge.m 中的 handleBuffer 方法，具体代码如图3所示。在 handleBuffer 中针对每个组件使用一个 queue 来处理对应任务。其中，这个 queue 是组件数据 RCTModuleData 中的属性 methodQueue，后文会详细介绍。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dcd0e879f.png" alt="图3  handleBuffer方法调用" title="图3  handleBuffer方法调用" />

图3  handleBuffer 方法调用

从上面的代码追踪可以看出，虽然 JS 只具备单线程操作的能力，但通过利用 Native 端多线程处理能力，仍可以很好地处理RN中的任务。回到刚开始抛出的议题，RN 在这里用 GCD 而非 NSOperationQueue 来处理线程，笔者认为主要原因有：

- GCD 更加轻量，更方便与 Block 结合起来进行线程操作，性能上优于 NSOperationQueue 的执行；

- 虽然 GCD 在控制线程数上有缺陷，不如 NSOperationQueue 有直接的 API 可以控制最大并发数，但由于JS是单线程发起任务，在5 ms 内会积累的任务数创造的并发不高，不用考虑最大并发数带来的CPU性能问题。

- 关于线程依赖的处理，由于 JS 端是在同一个线程顺序执行任务的，而在 Native 端对这些任务进行了分类（后文会有叙述），针对同类别任务在同一个 FIFO 队列中执行。这样的应用场景及 Native 端对任务的分类处理，规避了线程依赖的复杂处理。

#### 组件中线程自定义

前文提到了 Native（iOS）端处理并发任务的线程是 RCTModuleData 中的属性 methodQueue。RCTModuleData 是对组件对象的实例（instance）、方法(methods)、所属线程（methodQueue）等方面的描述。每一个 module 都有个独立的线程来管理，具体线程的初始化在 RCTModuleData 的 setUpMethodQueue 中进行设置，详细代码可见图4。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dcda8f442.png" alt="图4  线程自定义" title="图4  线程自定义" />

图4  线程自定义

图4中的174行至177行是开放给组件自定义线程的接口。如果组件实现了 methodQueue 方法，则获取此方法中设置的 queue；否则默认创建一个子线程。问题来了，既然可以自定义线程，那 RN 中内置组件是如何定义的，对开发过程中的自定义组件在设置线程的时候需要注意什么？

图5是本地项目中实现 methodQueue 的组件，除去以 RCTWB 开头的自定义组件，其它都是系统自带的。通过查看每一个组件 methodQueue 方法的实现，发现有的是在主线程执行，有的是在 RCTJSThread 中执行，表1所示的是其中主要系统组件的具体情况。

表1  RCTJSThread 中的主要系统组件一览果

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840e24e74761.jpg" alt="表1  RCTJSThread中的主要系统组件一览果" title="表1  RCTJSThread中的主要系统组件一览果" />

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dceb8df25.png" alt="图5  本地项目中实现methodQueue" title="图5  本地项目中实现methodQueue" />

图5  本地项目中实现 methodQueue

#### RCTJSThread

RCTJSThread 是在 RCTBridge 中定义的一个私有属性，如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dcfba3f0c.png" alt="图6  RCTJSThread定义" title="图6  RCTJSThread定义" />

图6  RCTJSThread 定义

RCTJSThread 的类型是 dispatch_queue_t，它是 GCD 中管理任务的队列，与 block 联合起来使用。一个 block 封装一个特定的任务，dispatch_queue_t 一次执行一个 block，相互独立的 dispatch_queue_t 可以并发执行 block。

RCTJSThread 的初始化比较有意思，并没有采用 dispatch_queue_create 来创建一个 queue 实例，而是指向 KCFNull。我在整个源代码里全局搜了一下，没有其他的地方对 RCTJSThread 进行初始化。事实上，RCTJSThread 在设计上不是用来执行任务的，而是用来进行比较的，看图7中的代码。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd0e0c150.png" alt="图7  RCTJSThread设计" title="图7  RCTJSThread设计" />

图7  RCTJSThread 设计

RCTBatchedBridge.m 中的 handleBuffer 是处理 JS 向 Native 端的事件请求的。在第928行，如果一个组件中定义的 queue 是 RCTJSThread，则在 JSExecutor 中执行 executeBlockOnJavaScriptQueue:方法，具体执行代码如图8所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd1b86296.png" alt="图8  执行executeBlockOnJavaScriptQueue:方法" title="图8  执行executeBlockOnJavaScriptQueue:方法" />

图8  执行 executeBlockOnJavaScriptQueue:方法

_javaScriptThread 是一个 NSThread 对象，看到这里才知道真正具备执行任务的是这里的 JavaScriptThread，而不是前面的 RCTJSThread。在 handBuffer 方法中之所以用 RCTJSThread，而不用 nil 替代，我的看法是为了可读性和扩展性。可读性是指如果在各个组件中将当前线程对象设置为 nil，使用者会比较迷惑；扩展性是指如果后面业务有扩展，发现根据 nil 比较不能满足需求，只需修改 RCTJSThread 初始化的地方，业务调用的地方完全没有感知。

#### RCTUIManagerQueue

RN的UI组件调用都是在 RCTUIManagerQueue 完成的，关于它的创建如图9所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd2496de3.png" alt="图9  创建RCTUIManagerQueue代码" title="图9  创建RCTUIManagerQueue代码" />

图9  创建 RCTUIManagerQueue 代码

由于苹果在 iOS 8.0 之后引入了 NSQualityOfService，淡化了原有的线程优先级概念，所以 RN 在这里优先使用了8.0的新功能，而对8.0以下的沿用原有的方式。但不论用哪种方式，都保证 RCTUIManagerQueue 在并发队列中优先级是最高的。到这里或许有疑问了，UI 操作不是要在主线程里操作吗，这里为什么是在一个子线程中操作？其实在此执行的是 UI 的准备工作，当真正需要把 UI 元素加入主界面，开始图形绘制时，才需要在主线程里操作，具体代码见图10。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd34be0cd.png" alt="图10  UI操作代码" title="图10  UI操作代码" />

图10  UI 操作代码

这里 RCTUIManagerQueue 是一个先进先出的顺序队列，保证了 UI 的顺序执行不出错，但这里是把 UI 的一些需要准备的工作（比如计算frame）放在一个子线程里面操作完成后，再统一提交给主线程进行操作的。这个过程是阻塞的，针对一些低端机型渲染复杂界面，会出现打开 RN 页面的一段空白页面的情况，这是 RN 需要优化的一个地方。

前面介绍了组件中线程的相关情况，针对平常开发中的自定义组件，有以下两点需要关注：

- 如果不通过 methodQueue 方法设定具体的执行队列(dispatch_queue_t)，则系统会自动创建一个默认线程，线程名称为ModuleNameQueue；

- 对同类别组件进行划分，采用相同的执行队列（比如系统 UI 组件都是在 RCTUIManagerQueue 中执行）。这样有两点好处，一是为了控制组件执行队列的无序生长，二也可以控制特殊情况下的线程并发数。

### JSBundle 加载中的线程操作

前面叙述的组件相关的线程情况，从业务场景方面来看，略显简单，下面将介绍一下场景复杂点的线程操作。

React Native 中加载过程业务逻辑比较多，需要先将 JSBundle 资源文件加载进内存，同时解析 Native 端组件，将组件相关配置信息加载进内存，然后再执行 JS 代码。图11所示的 Native 端加载过程代码，在 RCTBatchedBridge.m 的 start 方法中。其中片段1是将 JSBundle 文件加载进内存，片段2是初始化 RN 在 Native 端组件，片段3是设置 JS 执行环境以及初始化组件的配置，片段4是执行 JS 代码。这4个代码片段对应4个任务，其中任务4依赖任务1/2/3，需要在它们全部执行完毕后才能执行。任务1/3可以并行，没有依赖关系。任务3依赖任务2，需要任务2执行完毕后才能开始执行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd42dd9a4.png" alt="图11  Native端加载过程代码" title="图11  Native端加载过程代码" />

图11  Native 端加载过程代码

为控制任务4和任务1/2/3之间的依赖关系，定义了 dispatch_group_t initModulesAndLoadSource 管理依赖；而任务3依赖任务2是采取阻塞的方式。下面分别看各个任务中的处理情况。

先看片段1的代码，如图12所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd53506dd.png" alt="图12  片段1代码" title="图12  片段1代码" />

图12  片段1代码

```
dispatch_group_enter(group);
dispatch_async(queue, ^{
　　dispatch_group_leave(group);
});
```

但这里并没有使用 dispatch_async，而是采用默认的同步方式。具体原因在于 loadSource 中有一部分属性是下一个队列需要使用到的，这部分属性的初始化需要在这个队列中进行阻塞的同步执行。LoadSource 方法中有一部分逻辑是异步的，这部分数据可以在 initModulesAndLoadSource 的 group 合并的时候处理。

片段2的处理比较简单，跳过直接看片段3的代码，如图13所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd6431d4c.png" alt="图13  片段3代码详情" title="图13  片段3代码详情" />

图13  片段3代码详情

片段3中的任务是又一个复合任务，由一个新的 group（setupJSExecutorAndModuleConfig）来管理依赖。有两个并发任务，初始化 JS Executor（119-124行）和获取 module 配置（127-133行）。这两个并发任务都放在并发队列 bridgeQueue 中执行，完成后进行合并处理（135-150行）。需要注意的是片段3中采用 dispatch_group_async(group, queue, ^{ });来执行队列中的任务，其效果与前文叙述的 dispatch_group_enter／dispatch_group_leave 相同。

从上面的分析可以看出，GCD 利用 dispatch_group_t 可以很好地处理线程间的依赖关系。里面的线程操作虽不能像前文中组件的线程对开发有直接帮助，但是一个很好的利用 GCD 解决复杂任务的实例。

### 图片中的线程

看过 SDWebImage 的源码的同学知道，SDWebImage 采用的是 NSOperationQueue 来管理线程。但是 RN在image 组件中并没有采用 NSOperationQueue，还是一如继往地使用 GCD，有图14为证。眼尖的同学会发现图中明明有一个 NSOperationQueue 变量 _imageDecodeQueue，这是干什么用的？有兴趣可以在工程中搜索一下这个变量，除了在这里定义了一下，没有在其他任何地方使用。

我猜当时作者是不是也在纠结要不要使用 NSOperationQueue，而决定用 GCD 之后忘了删掉这个变量。

既然决定了使用 GCD，就需要解决两个棘手的问题，控制线程的并发数以及取消线程的执行。这两个问题也是 GCD 与 NSOperationQueue 进行比较时谈论最多的问题，且普遍认为当有此类问题时，需要弃 GCD而选 NSOperationQueue。下面就来叙述一下 RN 中是如何来解决这两个问题的。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dd7835f21.png" alt="图14  NSOperationQueue or GCD?" title="图14  NSOperationQueue or GCD?" />

图14  NSOperationQueue or GCD?

#### 最大并发数的控制

首先是控制线程的并发数。在 RCTImageLoader 中有一个属性 maxConcurrentLoadingTasks，如图15所示。除此之外，还有一个控制图片解码任务的并发数 maxConcurrentDecodingTasks。加载图片和解码图片是一项非常耗内存/CPU 的操作，所以需要根据业务需求的具体情况来灵活设定。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840ddb318eed.png" alt="图15  maxConcurrentLoadingTasks属性" title="图15  maxConcurrentLoadingTasks属性" />

图15  maxConcurrentLoadingTasks 属性

控制线程的最大并发数的逻辑在 RCTImageLoader的dequeueTasks 方法中，如图16所示。并发任务存储在数组_pendingTasks 中，当前进行中的任务数存储在_activeTasks 中。由于不能像 NSOperationQueue 中的任务一样，执行完毕后就被自动清除，在这里需要手动清除已经执行完毕的任务，将任务从_pendingTasks 中移除，并改变并发任务数，具体在代码的240行至251行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840ddc1a041a.png" alt="图16  dequeueTasks方法定义" title="图16  dequeueTasks方法定义" />

图16  dequeueTasks 方法定义

接下来看控制任务的执行代码，在267行至276行。遍历需要执行的任务数组，如果规定的条件超过了最大并发任务数，中断操作；否则直接执行任务，同时将计数器加1。由于所有任务都是在_URLCacheQueue 这个顺序队列中执行的，且一次只能执行一个任务，所以并发的实现是在 RCTNetworkTask 中进行的，有兴趣的同学可以深入看看。

进展到这里，最大并发数的控制还有一个关键任务没有完成，就是如何保证加入队列中的任务能全部完成。具体操作分为三个方面：首先是在加载图片的入口方法中有一次调用，如图17所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840ddd290cd9.png" alt="图17  图片加载入口方法中的调用" title="图17  图片加载入口方法中的调用" />

图17  图片加载入口方法中的调用

其次，需要处理等待任务。比如当前队列中已经有了最大并发数个任务了，下一个任务过来的时候只能暂时加入队列等待了。如果后续没有事件来调用 dequeueTasks 方法，超过最大并发数之外的任务将会得不到执行。一个通用的做法是用一个定时器来维持，定时扫描任务队列来执行任务。但是 RN 里面借助了图片渲染的逻辑巧妙地避开了这个，即在解码完成时调用一次 dequeueTasks 方法，这时候能保证等待任务能全部执行完毕，具体如图18所示代码。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840dde2aee0b.png" alt="图18  解码完成，调用dequeueTasks方法" title="图18  解码完成，调用dequeueTasks方法" />

图18  解码完成，调用 dequeueTasks 方法

最后，还有一种情况，即在线程取消的时候也需要调用一次 dequeueTasks 方法，来保证线程取消的情况下任务也能继续完成。这样综合上述三种情况的调用，加入队列中的任务都能全部执行完毕了。

#### 线程的取消

如果说上面的最大并发数的控制还可以有方法自定义实现，但是线程的取消一直是 GCD 中无法做到的，只能通过 NSOperation 的接口来实现。上文提到了加载图片并发操作是在 RCTNetworkTask 中实现的，而 RCTNetworkTask 调用的是 RN 中 RCTNetwork 中的代码，先来简单介绍一下 RCTNetwork 的实现。

图19所示的是与图片访问相关的实现类 RCTDataRequestHandler。可以看出，数据访问的任务是用 NSOperationQueue 来管理的，线程的取消是调用 NSOperation的cancel 方法来执行的。后面介绍到的图片下载任务的取消即基于此。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840ddfa554bf.png" alt="图19  RCTDataRequestHandler实现" title="图19  RCTDataRequestHandler实现" />

图19  RCTDataRequestHandler 实现

回到图片组件中的线程取消上来。在 RCTImageLoader 的加载/解码图片的方法中返回参数为任务取消 block：RCTImageLoaderCancellationBlock。Block 的具体实现在每一个方法中，以 loadImageWithURLRequest 为例，如图20所示。[task cancel]调用的是上述的 NSOperation 的 cancel 方法。

<img src="http://ipad-cms.csdn.net/cms/attachment/201612/5840de10776b9.png" alt="图20  取消加载" title="图20  取消加载" />

图20  取消加载

至此，RN 在优先使用 GCD 的情况下，完成了图片组件中的线程相关逻辑。还是回到最开始讨论的话题，GCD 在控制任务状态，比如取消、悬挂、等待、监控线程等，目前采取自定义方法没有很好的方式实现，还得借助于 NSOperation。而在控制最大并发数方面，RN 提供了一个很好的自定义实现的例子，值得学习。

### 结语

本文从组件、JSBundle 加载、图片中的线程三个方面，对 RN 的源代码实现，以具体的实例，叙述了 RN 中线程管理的详细情况。这三个例子，从技术实现上，复杂度逐步增加，覆盖了线程中任务依赖、最大并发数的控制、线程取消等经典讨论点。特别是在任务依赖、最大并发数的控制上，给我们呈现了用 GCD 来解决的一个很好的实例。

参考资料

1.《Choosing Between NSOperation and Grand Central Dispatch》，链接：https://cocoacasts.com/choosing-between-nsoperation-and-grand-central-dispatch/；

2.《NSOperation vs Grand Central Dispatch》，链接：http://stackoverflow.com/questions/10373331/nsoperation-vs-grand-central-dispatch；

3.《Is JavaScript guaranteed to be single-threaded》，链接：http://stackoverflow.com/questions/2734025/is-javascript-guaranteed-to-be-single-threaded；

4.《浅析 RN 通讯机制》，链接：http://www.jianshu.com/p/203b91a77174。