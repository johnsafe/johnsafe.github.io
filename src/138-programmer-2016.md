## React Native：下一代移动开发框架？

文/黄文臣

React Native 是一个能够使用 JavaScript 和 React 来开发 iOS 或 Android App 的框架。随着 Facebook 在2015年 React.js Conf 大会上推出并开源了 React Native，这个框架受到业界关注，越来越多的公司开始尝试和使用。

目前，React Native 在 GitHub 上共收到了34000+ star，7400+ fork。使用 React Native，可以像 Web 开发获得快速迭代反馈的能力；同时，也可以获得 Native 开发的用户体验，并且可以使用 React 在 Web 端已经验证成功的框架。

### 移动开发的瓶颈

一个框架的出现必然是为了解决某些问题。正是因为移动端开发的诸多瓶颈，才催生了 React Native。

#### 布局困难

对于 iOS 开发，开发者可以使用 AutoLayout 来进行布局。AutoLayout 是一个非常适合 Xib 和 Storyboard 的技术，因为开发者可以预览视图变化。但是，多人团队很少使用 Xib 或者 Storyboard。因为这样的文件本质是大 XML，最终会造成多人同时修改，代码合并困难。因此，开发者不得不使用代码来实现复杂的约束，虽说有 Masonry 这样的优秀开源库，但是仍然没有本质上解决布局困难的问题。

对于 Android 开发，Google 在设计之初，借鉴了 Web 的布局方式，开发者可以采用 XML 文件的方式布局。但是，这种布局方式仍然没有 CSS 灵活，Google 推出了 Android 版本 Flexbox-layout 开源库，一定程度上解决了这个问题。

#### 代码复用的问题

Android 开发主要使用 Java，iOS 开发使用 Objective-C 或者 Swift。语言、开发和编译环境的不同导致即使是一模一样的界面，Android 和 iOS 也需要各写一套代码，进而需要两次测试。为此，微软曾推出了 Xamarin 框架来统一 Android 和 iOS 开发，但是 Xamarin 的诸多问题导致其并没有被大多数开发者采用。

#### 迭代周期长，难以抵达用户

对于 Web 开发，假如在服务器端修改了 HTML 或者 JavaScript，用户刷新就可以获得实时的反馈。但是，移动端则不行。开发者不得不重新编译→打包→测试→提交应用商店，并且只有用户选择更新，才能让新的特性抵达用户。这意味着 A/B 测试的周期会更长，从而导致即使公司有足够的开发资源，也难以提高迭代的速度。

#### 人才稀缺与企业成本

现在 iOS/Android 培训班很多，也从侧面反映出移动开发行业对这类人才的需求。虽说移动开发者的基数很多，但是高水平的开发者却很少。iOS 开发还需要 Objective-C 或者 Swift 语言的掌握。而大学期间，这两类语言很少有学校的课程会涉及。由于 React Native 的主要语言是 JavaScript，其普及和使用程度要更高，也更易上手。同样一个 App，企业可能不再需要一个 Android，一个 iOS，以后只要一个 React Native 开发者就够了。

### Hybrid App

因为移动开发的种种缺陷，Hybrid App 诞生了，其中比较有名的框架有 Ionic 和 PhoneGap。这类框架的特点是把页面嵌入到 WebView 里运行，必要的性能较高的页面用 Native 来是实现。这样做有诸多好处：

1. 某些界面能够动态部署；

2. 由于是运行在 WebView 里的，这时候 HTML5、JavaScript、CSS 所有 Web 相关的技术都可以用，并且 Ionic 还提供了很多现成的组件，布局也变得很灵活；

3. 代码可以复用，因为基于 Web，Android 和 iOS 上几乎可以无缝复用。

但是，有经验的开发者都知道，这种框架的缺点也非常明显：在 WebView 里运行的界面永远无法获得 Native 一样的用户体验。

于是，进行 Hybrid 开发，大部分页面还是要 Android 和 iOS 分开来开发。只对于那些纯展示性，交互简单，界面复杂的采用 WebView 嵌套方式。

### 工作原理

本文主要基于 iOS 平台最新的 React Native v0.29.1源代码阅读和理解。由于 Reac Native 是一个相当庞大的库，本文只能管中窥豹。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579f0635b3b28.png" alt="图1  Native和JavaScript调用的核心枢纽" title="图1  Native和JavaScript调用的核心枢纽" />

图1  Native 和 JavaScript 调用的核心枢纽

整个 Native 和 JavaScript 调用的核心枢纽如图1所示，其中：

1. Objective C Bridge 主要维护 JS Engine 和 Native Module。JS Engine 主要负责执行 JavaScript 的代码和把 Native 代码执行的结果回调给 JavaScript。Native Module 维护几个数组，用来存储 Native 暴露给 JavaScript 的类和方法；

2. JS Bridge 主要维护 JS Module、Native Modules 和 Message Queue。其中 JS Module 和 Native Modules 分别维护了 JavaScript 暴露给 Native 的和 Native 暴露给 JavaScript 的接口。Message Queue 负责处理 JavaScript 调用 Native 的队列，以及负责分发 Native 调用 JavaScript 的请求。

#### 模块暴露

App 启动时，iOS 会获取所有暴露给 JavaScript 的类和对应的方法，存储到 Obective C Bridge 中。然后把这些类和方法以 JSON 格式的字符串注入到 JavaScript 中，JSON 字符串存储的内容称作 Module Config。JavaScript 有一个全局变量，用来存储这个 Module Config，并且动态合成对象。这样，Native 的类和方法就暴露给了 JavaScript。

注入的 JSON 格式如下：

```
{
     “remoteModuleConfig”:[
        …
        [  
               “RCTActionSheetManager", 
             [  
                  "showActionSheetWithOptions",
                  "showShareActionSheetWithOptions"
             ]
         ],
       …
      ]
}
```

JavaScript 把模块方法定义成懒加载的方式，只有在需要调用的时候，才会调用构造函数，把对应模块注册到 JS Bridge 中的 JS Modules。然后，Native 通过 JavaScriptCore 来调用 JS Bridge 一个统一的方法，再由这个方法进行分发，找到实际的执行体。

#### 通信原理

Native 调用 JavaScript 比较简单，分为以下几步：

1. 通过 Objective-C Bridge 中的 JS Engine 调用 JavaScript 方法，传递对象名、方法名、参数、回调；

2. JS Bridge 通过对象名，方法名从 JS Module 中查询，找到对应的方法；

3. 如果对应的方法还没加载，则加载对应的方法，然后执行对应的方法，执行回调。

JavaScript 调用 Native 的代码是通过 ID 来映射的。这些 ID 包括 ModuleID、MethodID、CallBackID。其中：

1. ModuleID 表示模块 ID，比如 ID 是35表示 RCTActionSheetManager，它是封装了 iOS Native ActionSheet 的模块；

2. MethodID 表示模块中方法的 ID；

3. CallBackID 表示执行完 Objective-C 方法后，回调给 JavaScript 的代码块 ID。

上文提到的 JS Bridge 和 Objective C bridge 的一个很重要的任务，就是来生成和维护这些 ID。

```
ActionSheetIOS.showActionSheetWithOptions({
      options: ["取消","确定"],
      cancelButtonIndex: 0,
      destructiveButtonIndex: 1,
    },
    (buttonIndex) => {
      console.log("Click index " + buttonIndex);
    });
```

举个例子，用 JavaScript 来调用系统 NativeActionSheet，则整个调用过程如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579f074404b4d.png" alt="图2  调用过程" title="图2  调用过程" />

图2  调用过程

#### 事件循环

React Native 模拟 Native App 事件循环机制，在 iOS 中也就是 Runloop 机制。只在事件（触摸、Timer、System 事件）到来的时候，执行代码。比如，在接收到触摸的时候，Native 会实时地把触摸相关信息传递给 JavaScript。这也就是为什么 React Native 在接受触摸方面和 Native 几乎没有差别的原因。

在事件循环中，一个循环周期是5 ms，在这5 ms 的 JavaScript 调用 Native 的代码，会缓存起来，通过返回值一起返回给 Native。这样做，降低了 JavaScript 和 Native 的调用频率，也提高了整体运行的性能。

#### 布局原理

React Native 的布局是通过 FlexBox 来实现的。不难理解，这个布局最后必然会转换成 frame(x, y, width, height)，Native 的代码才知道如何放置 View。

我们先来看一个例子：这是一个最简单的视图：

```
<view style="{{flex:" 1,justifycontent:="" 'center',="" alignitems:="" 'center'}}="">
    <text>一个测试标题</text>
</view>
```

在渲染这个视图的时候会进行如下调用：

```
JS->N : RCTUIManager.createView([2,"RCTView",1,{"flex":1}])
JS->N : RCTUIManager.createView([3,"RCTView",1,{"flex":1}])
JS->N : RCTUIManager.createView([4,"RCTView",1,{"flex":1,"justifyContent":"center","alignItems":"center"}])
JS->N : RCTUIManager.createView([5,"RCTText",1,{"accessible":true,"allowFontScaling":true,"lineBreakMode":"tail"}])
JS->N : RCTUIManager.createView([6,"RCTRawText",1,{"text":"一个测试标题"}])
JS->N : RCTUIManager.setChildren([5,[6]])
JS->N : RCTUIManager.setChildren([4,[5]])
JS->N : RCTUIManager.setChildren([3,[4]])
JS->N : RCTUIManager.createView([7,"RCTView",1,{"position":"absolute"}])
JS->N : RCTUIManager.setChildren([2,[3,7]])
JS->N : RCTUIManager.setChildren([1,[2]])
```

所以，布局的原理也很清楚：

1. 通过 React 的 render()方法来返回视图的层级关系和对应的 Style；

2. 将对应的 Style 和层级关系，生成对应的配置 JSON，这个 JSON 存储了视图的 ID、参数、flexBox 属性等信息；

3. 通过 JS Bridge，将对应的参数和层级关系通过 RCTUIManager 这个枢纽传递 Native；

4. Native 对收到的 View 层级和 JSON 进行解析，解析成 frame、backgroundColor 等参数；

5. 进行视图布局和渲染。

### 优势

#### 动态部署

动态部署是 React Native 最吸引笔者的优点。有了动态部署，开发的迭代周期就会更短；即使上线的版本有 Bug，也可以在运行时动态修复，不用担心用户不升级，新的功能抵达不了。运营团队也可以更加灵活地安排各种活动，App 可以及时的按照需求进行增加/删除活动页面。

#### 接近 Native 的交互体验

由于 React Native 没有借助 WebView 这个壳子来运行代码，而是自己实现了一套 Native 和 JavaScript 通信的机制。所以，在触摸、手势、动画上可以获得和 Native 几乎没有差别的体验。

#### 强大的可扩展性

只要开发者遵循 React Native 的 JavaScript 和 Native 通信原理，就可以在 Native 中封装复杂的代码，然后用 JavaScript 调用。同时，它也支持用纯 JavaScript 来编写 iOS 和 Android 通用的代码。目前 GitHub 上，这种扩展库已达到几百个，笔者也建立了一个仓库《ReactNativeMaterials》来搜集这些开源库和博客，感兴趣的同行可以一起把它完善。

#### 其他优势

除了以上提到的几点，React Native 还有如下优势：

1. 代码复用。对于常见的 MVC 架构，在 JavaScript 中 Model 和 Controller 层几乎可以实现全部复用，在 View 层，也有诸如 Navigator 等许多平台无关的可复用视图。

2. 布局灵活。基于 FlexBox 的布局和基于 JavaScript 的 style，让布局变得前所未有的简单。

3. 优秀人才更多。通常，熟练 JavaScript 和 React 的开发者，可以很快就上手开发 React Native 代码。

### 目前存在的问题

React Native 完美吗？当然不。React Native 现在保持了两周更新一个版本的速度，不断迭代，修复问题，增加功能。目前看来，React Native 存在的问题也不少：

1. 平台相关的代码仍然很多。比如 PickerIOS、ProgressBarAndroid。对于 View 层，开发者在很多场景仍然不得不为两个平台写两套代码。

2. 多线程的处理能力。用 React Native 开发的代码，大部分逻辑是运行在 JavaScript 线程中的。对于 Native 开发，开发者可以合理的利用多核多线程，来处理复杂的渲染和计算，而显然在一个 JavaScript 线程实现这些不现实。

3. 缺少复杂的交互能力。由于 React Native 需要把触摸等交给 JavaScript 处理，然后 JavaScript 再回调。对于复杂的基于触摸的交互，React Native 显得力不从心。这种模块，通常需要纯 Native 来实现。

4. Android 性能问题。对于 iOS 设备来说，性能往往较好，除了一些复杂的界面，实现60帧并非难事。而 Android 机的性能差别比较大，并且 React Native 早期只支持 iOS，也就导致了发展至今，对安卓的支持，尤其是触摸方面，处理起来要比 iOS 棘手的多。

5. 社区还不完善。由于 React Native 仍然是一个新的框架（不到两岁），并且在不断更新，所以社区的成长速度要远低于新版本的发行速度。现在遇到的部分问题，很多开发者只能做第一个吃螃蟹的人，自己去解决。

### 结束语

React Native 无疑是个划时代的框架，国内外诸多的开发者汇聚一心共同完善它，同时越来越多的公司开始尝试将 React Native 应用到实际的产品中。鉴于 React 在 Web 端的成功，我们有理由相信，React Native 在 Mobile 的前景一片光明。当然，Facebook 出品并不一定代表会成功，至于它的未来究竟如何，我们一起见证。