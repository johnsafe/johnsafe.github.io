## iOS 动态更新方案 JSPatch 与 React Native 的对比

文/陈振焯

JSPatch 是 iOS 平台上的一个开源库，只需接入极小的三个引擎文件，即可用 JavaScript 调用和替换任意 Objective-C 方法。也就是说可以在 App 上线后通过下发 JavaScript 脚本，实时修改任意 Objective-C 方法的实现，达到修复 Bug 或动态运营的目的。目前 JSPatch 被大规模应用于热修复（Hotfix），已有超过2500个 App 接入。

虽然 JSPatch 目前大部分只用于热修复，但因为 JSPatch 可以调用任意 Objective-C 方法，实际上它也可以做热更新的工作。也就是动态为 App 添加功能模块，并对这些功能模块进行实时更新，可以起到跟 React Native 一样的作用。我们从学习成本、接入成本、开发效率、热更新能力和性能体验这几个方面来对比使用 React Native 和 JSPatch 做热更新的差异。

### 学习成本

React Native 是从 Web 前端开发框架 React 延伸出来的解决方案，主要解决的是 Web 页面在移动端性能低的问题。React Native 让开发者可以像开发 Web 页面那样用 React 的方式开发功能，同时框架会通过 JavaScript 与 Objective-C 的通信让界面使用原生组件渲染，让开发出来的功能拥有原生 App 的性能和体验。

这里会有一个学习成本的问题，大部分 iOS 开发者并不熟悉 Web 前端开发。当他们需要一个动态化的方案开发一个功能模块时，若使用 React Native，就意味着需要学习 Web 前端的一整套开发技能，学习成本很高。所以目前一些使用 React Native 的团队里，这部分功能是由前端开发者负责，而非终端开发者。

但前端开发者负责这部分功能也会有一些学习成本的问题，因为 React Native 还未十分成熟，出了 Bug 或有性能问题需要深入 React Native 客户端代码排查和优化。也就是说，React Native 是跨 Web 前端开发和终端开发的技术，要求使用者同时有这两方面能力才能使用得当，这不可避免地带来学习成本的提高。

而 JSPatch 是从终端开发出发的一种方案，写出来的代码风格与 Objective-C 原生开发一致，使用者不需要有 Web 前端的知识和经验，只需要有 iOS 开发经验，再加上一点 JavaScript 语法的了解，就可以很好地使用，对终端开发来说学习成本很低。

可以看一下同样实现一个简单的界面，React Native 和 JSPatch 代码的对比：

```
//React Native
class HelloWorld extends Component {
  render() {
    return (
      
        
          
    	function path()
		{
		  var args = arguments,
		      result = []
		      ;
		       
		  for(var i = 0; i < args.length; i++)
		      result.push(args[i].replace('@', '/cms/js/syntax/scripts/'));
		       
		  return result
		};
		 
		SyntaxHighlighter.autoloader.apply(null, path(
		  'applescript            @shBrushAppleScript.js',
		  'actionscript3 as3      @shBrushAS3.js',
		  'bash shell             @shBrushBash.js',
		  'coldfusion cf          @shBrushColdFusion.js',
		  'cpp c                  @shBrushCpp.js',
		  'c# c-sharp csharp      @shBrushCSharp.js',
		  'css                    @shBrushCss.js',
		  'delphi pascal          @shBrushDelphi.js',
		  'diff patch pas         @shBrushDiff.js',
		  'erl erlang             @shBrushErlang.js',
		  'groovy                 @shBrushGroovy.js',
		  'java                   @shBrushJava.js',
		  'jfx javafx             @shBrushJavaFX.js',
		  'js jscript javascript  @shBrushJScript.js',
		  'perl pl                @shBrushPerl.js',
		  'php                    @shBrushPhp.js',
		  'text plain             @shBrushPlain.js',
		  'py python              @shBrushPython.js',
		  'ruby rails ror rb      @shBrushRuby.js',
		  'sass scss              @shBrushSass.js',
		  'scala                  @shBrushScala.js',
		  'sql                    @shBrushSql.js',
		  'vb vbnet               @shBrushVb.js',
		  'xml xhtml xslt html    @shBrushXml.js'
		));
		SyntaxHighlighter.all();
    
    

```

代码1

```
//JSPatch
require('UIColor, UIScreen, UIButton')
defineClass('HelloWord : UIView', {
    initWithFrame: function(frame) {
        if(self = super.initWithFrame(frame)){
            var screenWidth = UIScreen.mainScreen().bounds().width
            var loginBtn = UIButton.alloc().initWithFrame({x: 20, y: 50, width: screenWidth - 40, height: 30});
            loginBtn.setBackgroundColor(UIColor.greenColor())
            loginBtn.setTitle_forState("Login", 0)
            loginBtn.layer().setCornerRadius(5)
            loginBtn.addTarget_action_forControlEvents(self, 'handleBtn', 1<<6);
            self.addSubview(loginBtn);
        }
        return self;
    },
    handleBtn: function() {
    }
})
```

代码2

### 接入成本

接入成本上，React Native 是比较大的框架，据统计目前核心代码里 Objective-C 和 JavaScript 代码加起来有4 w 行，接入后安装包体积增大1.8 MB 左右。而 JSPatch 是微型框架，只有3个文件2 k 行代码，接入后增大100 K 左右。另外 React Native 需要搭建一套开发环境，有很多依赖库，环境搭建是一个痛点。而 JSPatch 无需搭建，只需要拖入三个文件到工程中即可使用。

React Native 是大框架，维护起来成本也会增大，在性能调优和 Bug 查找时，必须深入了解整个框架的原理和执行流程。此外 React Native 目前还未达到稳定状态，升级时踩坑不可避免。相对来说 JSPatch 接入后的维护成本会低一些，因为 JSPatch 只是作为很薄的一层转接口，没有太多规则和框架，也就没有太多坑，本身代码量小，需要深入了解调试 Bug 或性能调优时成本也低。

### 开发效率

在 UI 层上目前 HTML+CSS 的方式开发效率是比手写布局高的，React Native 也是用近似 HTML+CSS 去绘制 UI。这方面开发效率相对 JSPatch 会高一些，但 JSPatch 也可以借助 iOS 一些成熟的库提高效率，例如使用 Masory，让 UI 的开发效率不会相差太多。逻辑层方面的开发效率双方一样。

此外，React Native 在开发效率上的另一个优势是支持跨平台。React Native 本意是复用逻辑层代码，UI 层根据不同平台写不同的代码，但 UI 层目前也可以通过 ReactMix 之类的工具做到跨平台，所以 UI 层和逻辑层代码都能得到一定程度的复用。而 JSPatch 目前只能用于 iOS 平台，没有跨平台能力。

实际上跨平台有它适用和不适用的场景，跨平台有它的代价，就是需要兼顾每个平台的特性，导致效果不佳。

跨平台典型的适用场景是电商活动页面，以展示为主，重开发效率轻交互体验，但不适用于功能性的模块。对 Android 来说目前热更新方案相当成熟，Android 十分自由，可以直接用原生开发后生成 diff 包下发运行，无论是开发效率和效果都是最好的。所以若是重体验的功能模块，Android 使用原生的热更新方案，iOS 使用 JSPatch 开发，会更适合。

JSPatch 也做了一些事情尝试提高开发效率，例如做了 Xcode 代码提示插件 JSPatchX，让用 JavaScript 调用 Objective-C 代码时会出现代码提示。另外跟 React Native 一样有开发时可以实时刷新界面查看修改效果的功能，目前仍在继续做一些措施和工具提高开发效率。

### 热更新能力

React Native 和 JSPatch 都能对用其开发出来的功能模块进行热更新，这也是这种方案最大的好处。不过 React Native 在热更新时无法使用事先没有做过桥接的原生组件，例如需要加一个发送短信功能，就要用到原生 MessageUI.framework 的接口，若没有在编译时加上提供给 JavaScript 的接口，是无法调用到的。而 JSPatch 可以调用到任意已在项目里的组件，以及任意原生 framework 接口，不需要事先做桥接。在热更新能力上，相对来说 JSPatch 的能力和自由度更高。

### 性能体验

使用 React Native 和 JSPatch 性能会比原生差点，但都能得到比纯 HTML5 页面或 Hybrid 更好的性能和体验。

JSPatch 的性能问题主要在于 JavaScript 和 Objective-C 通信，每次调用 Objective-C 方法都要通过 OC Runtime 接口，并进行参数转换。Runtime 接口调用带来的耗时一般不会成为瓶颈，参数转换则需要注意避免在 JavaScript 和 Objective-C 之间传递大的数据集合对象。JSPatch 在性能方面也针对开发功能做了不少优化，尽力减少了 JavaScript 和 Objective-C 通信，GitHub 项目主页上有完整的小 App Demo（https://github.com/bang590/JSPatch/tree/master/Demo/DribbbleDemo），目前来看并没有碰到太多性能问题。

React Native 的性能问题会复杂一些，因为框架本身的模块初始化/React 组件初始化/JavaScript 渲染逻辑等会消耗不少时间和内存，这些地方若使用或优化不当都会对性能和体验造成影响。JavaScript 和 Objective-C 的通信也是一个耗性能的点，不过 React Native 优化得比较好，没有成为主要消耗点。

在性能和体验上，两者有不同的性能消耗点，从最终效果来看差别不大。

### 总结

总的来说，JSPatch 在学习成本，接入成本，热更新能力上占优。而 React Native 在开发效率和跨平台能力上占优（见表1），大家可以根据需求不同选用不同的热更新方案。JSPatch 目前仍在不断发展中，后续会致力于提高开发效率，完善周边支持，欢迎参与这个开源项目的开发。

表1  React Native 与 JSPatch 综合对比

<img src="http://ipad-cms.csdn.net/cms/attachment/201608/579efcfa74ae6.jpg" alt="表1  React Native与JSPatch综合对比" title="表1  React Native与JSPatch综合对比" />