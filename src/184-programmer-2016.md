## iOS 平台无障碍化利器——VoiceOver

文/王哲，闫石

### 前言

VoiceOver 是苹果“读屏”技术的名称，属于辅助功能的一部分。当用户在选择了指定交互元素时，VoiceOver 会读出这个元素的信息和使用提示，以帮助盲人进行人机交互。

比如：当你轻点导航栏中的“返回”按钮，VoiceOver 会告诉你“返回 按钮”。 图1为 VoiceOver 的点击响应效果，一目了然。

### VoiceOver 基本操作

在 VoiceOver 模式下的手势操作和平常不同，以下列举一些常用的功能，见表1。

表1  VoiceOver 模式下的手势操作

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c68c52b1323.png" alt="表1  VoiceOver模式下的手势操作" title="表1  VoiceOver模式下的手势操作" />

VoiceOver 还有许多为视障人士提供的辅助操作,可以到苹果官网上查看更详细的内容。

### VoiceOver 开发入门

有了以上的了解，本节将介绍 APP 应用如何支持 VoiceOver。

#### 标准控件进行 VoiceOver 开发

**AccessibilityElement 的概念**

AccessibilityElement，它是指可以被 VoiceOver 访问的 UI 元素，UIKit 中的控件基本都支持 VoiceOver，但 UIView 不是，不过也可以通过设置 UIView的isAccessibilityElement 属性来控制某个 View 是否是 AccessibilityElement，在 UIKit 的控件中，像 UILabel，UIButton 这些控件的 isAccessibilityElement 属性默认就是 YES，UIView 这个属性默认是 NO。

**AccessibilityElement 的应用**

一般的 AccessibilityElement 系统会自动读出它的内容，所以很多情况下不需要做特殊处理应用也会支持 VoiceOver。

**扩展应用**

但如果想自定义读出的内容就需要用到以下属性：

**accessibilityLabel（标签）**一个简单的词或短语，它简洁明了地描述控件或者视图，但是不能识别元素类型，UIAccessibilityElement 必须要有这个属性。例如“添加”、“播放”。

**accessibilityHint（提示）**一个简单的词或短语，描述发生在元素上动作的结果。例如“添加标题”或者“打开购物列表”。

**accessibilityValue（值）**不是由标签定义的元素时的当前值。仅当元素的内容是可改变并且不能使用 label 描述时，一个无障碍元素才需要为其赋值。例如，一个进度条的标签可以是”播放进度”，但是它当前的值是“50%”。

accessibilityTraits 这个 element 的类型以及状态，就是通过 traits 来表征这个 Element 的特质，数据类型是一个枚举类型，可以通过按位或的方式合并多个特性。

VoiceOver 会把这几个属性连接起来，朗读顺序为 label→value (可选)→traits→hint。但需要注意的是，当某个 View 的是 AccessibilityElement 的时候 ，其 subviews 都会被屏蔽掉，如果想要都读出来，只能改变他们的层次结构，并都设置 isAccessibilityElement 为 YES。这个特性还是挺有用的，比如一个 View 中有多个 Label，那么每一个下面的 Label 单独访问可能意义不大，那么就可以将这个 View 设置成可以访问的，然后将其 accessibilityLabel 设置为所有子 Label 的 accessibilityLabel 的合并值。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c68cac03eea.jpg" alt="图1  VoiceOver效果演示" title="图1  VoiceOver效果演示" />

图1  VoiceOver 效果演示

#### 自定义控件进行 VoiceOver 开发

这里的自定义控件特指 UIView 的某些子类，默认情况下自定义控件是不支持 VoiceOver 的。

**可以通过两种方式使之支持其功能：**

- 实现 isAccessibilityElement 协议，在 UIView 子类中实现此协议返回 YES。

- 直接设置 isAccessibilityElement 属性，将其设置为 YES。

**自定义其他属性和标准控件一样**

需要注意的是通常情况下控件在使用 VoiceOver 时 touch 事件会失效，但是有时候需要让控件处理 touch 事件，比如画图功能，画布需要处理触摸事件，这时就需要设置画布的 accessibilityTraits: paintCanvas.accessibilityTraits 

```
|= UIAccessibilityTraitAllowsDirectInteraction。
```

如果想要实现监听焦点是否在当前控件上，可以在子类中实现：

- accessibilityElementDidLoseFocus 方法。

- accessibilityElementDidBecomeFocused 方法。

- 再利用编写协议回调来实现监听焦点是否在当前控件上并执行特定操作。

例如：

```
- (void)accessibilityElementDidLoseFocus{
[self.delegate viewDidLoseFocus];
}
- (void)accessibilityElementDidBecomeFocused{
     [self.delegate viewDidBecomeFocused];
}
```

#### iOS 平台开放的其他 VoiceOver 关键能力

**通知**

Accessibility 提供了一系列的通知，可以完成一些特定的需求。

你可以监听 UIAccessibilityVoiceOverStatusChanged 通知，来监控 VoiceOver 功能开启关闭的实时通知 。也可以使用 UIAccessibilityPostNotification 方法在 App 中主动发送一些通知，来让系统做出一些变化，它有两个参数，第一个是通知名，第二个是你想让 VoiceOver 读出来的字符串或者是新的 VoiceOver 的焦点对应的元素。

比如：

你想让焦点转移到特定控件：

```
UIAccessibilityPostNotification(UIAccessibilityScreenChangedNotification,self.element);
```

或者读出一段语音

```
UIAccessibilityPostNotification(UIAccessibilityAnnouncementNotification, @"需要读出来的内容");
```

除此之外还有很多通知，可以去苹果官方文档查找更详细的说明。

**模拟器中调试 VoiceOver**

在模拟器中想要使用 VoiceOver 也需要在设置里打开，不过打开之后并不会读出声音，而是会出现一个提示框，点击左上角关闭按钮可以关闭 VoiceOver，再次点击重新打开，很方便，如图2所示：

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c68ebe6cb9a.png" alt="图2  VoiceOver的模拟器调试" title="图2  VoiceOver的模拟器调试" />

图2  VoiceOver 的模拟器调试

- Accessibility Inspector 是展示元素的可用性。

- Label 对应 accessibilityLabel 属性。

- Traits 对应 accessibilityTraits 属性。

- Frame 对应 accessibilityFrame 属性 （朗读时在屏幕上显示的聚焦框位置和大小，一般不需改动）。

### VoiceOver 开发进阶

前面提到过 VoiceOver 模式下有几种特殊的手势，相关的操作方法在上文有所介绍，这里不再赘述。

这些手势需要重写特定的方法来执行视图或视图控制器的特定任务。最基本的一条原理，即 UIKit 使用 responder 链搜索方法，从有 VoiceOver 焦点的元素开始。如果没有对象实现合适的方法，执行系统默认操作。

需要注意的是，所有的 VoiceOver 特殊手势方法都返回一个 BOOL 值，该值决定是否通过 responder 链传递。暂停传递返回 YES，否则返回 NO。

#### 双指搓擦（Z 字形手势）

使用 accessibilityPerformEscape 方法处理双指搓擦手势。

双指搓擦的手势像是电脑的 ESC 键的功能，它关闭一个临时对话框或浮层来回到原本界面。也可以使用双指搓擦手势在自定义导航层级中回到上一级页面。

如果已经使用了 UINavigationController 对象 ，那么不需要实现该手势，因为 UINavigationController 对象已经处理了该手势。

#### 魔法轻拍（双指双击）

使用 accessibilityPerformMagicTap 方法来处理魔法轻拍手势。

这个手势没有特定使用场景，它用来快速执行用户常用的或最想要的操作。比如在计时应用中用来开始/停止计时，或者在音乐播放器中用来切换歌曲等。

#### 三指滚动

使用 accessibilityScroll:处理三指滚动手势，滚动自定义视图的内容。它接受一个 UIAccessibilityScrollDirection 类型的参数，表示滚动的方向。

### 结束语

短暂的 VoiceOver 入门之旅已经接近终点，希望通过这篇文章能让大家对 VoiceOver 开发能有比较全面的认识。笔者这里只是介绍了较为基础的部分，还有如 UI Accessibility 编程接口或者非 UIView 和其子类的 UI 元素如何支持 VoiceOver 等等。学无止境，只有真正用心才能做出真正帮助到视障人群的无障碍化产品。
