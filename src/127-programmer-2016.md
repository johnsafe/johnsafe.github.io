## 揭秘 Android N 新的编译工具 JACK&JILL

文/李晓阳

>在 Android 5.0修订版 SDK 中，上线了两款新编译器，名为 JACK&JILL，它们象征着 Google 在进一步增强变异流程。JACK&JILL 集成了混淆、资源裁剪、Multidex 等原本以 Gradle 插件形势加入编译过程的工具，使用 jayce 作为 IR，彻底脱离了对 class 字节码文件的依赖。本文将对 JACK&JILL 进行深度剖析，并对其使用进行讲解。

近日，Android N 正式推出了全新的编译工具链——JACK&JILL（JACK，Java Android Compiler Kit；JILL，Java Intermediate Library Linker)，这一工具链早在21.1.0版本 Android build tools 即有内置，只是随着 Android N 预览版正式作为编译工具链推出，集成了混淆、资源裁剪、Multidex 等原本以 Gradle 插件形式加入编译过程的工具，使用 ecj 替代 Javac 作为Java源码编译工具，使用 jayce 作为 IR，彻底脱离了对.class 字节码文件的依赖。JACK&JILL 的原始的构建流程如图1所示。对比基于 JACK 的构建流程如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d39b93c4c6.png" alt="图1 Java + Proguard + dx的构建流程" title="图1 Java + Proguard + dx的构建流程" />

图1 Java + Proguard + dx 的构建流程

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d39c3bcdd0.png" alt="图2 对比基于JACK的构建流程" title="图2 对比基于JACK的构建流程" />

图2 对比基于 JACK 的构建流程

现有工程中依赖的大量库仍然是以传统 jar 包形式存在，而 JILL 在编译过程中将其转化为新的 JACK library，再由 JACK 统一进行编译混淆，资源优化，打包为 dex 文件。

Google 之所以要推出一套全新的编译工具链，个人猜测有以下几个原因：

- 目前的 Android 编译工具链是一个多方工具的组合，有开源的 Proguard、Android 自带的 dx、Oracle 的 Java 编译器。工具链的各种工具分散在互不相干的各方手中，演进缓慢且进度不一，大大影响了 Android 开发环境的改进。比如最近比较火的 RxJava 所需的 lambda 表达式（当然不使用 lambda 也可以，只是代码冗长，可读性会很差)，Java 直到1.8才添加了一个不怎么完善的 lambda 支持, 而 Android 自带的 dx 干脆就不支持 Java1.8的 class，这导致 Android 到开发者们不得不使用 Retrolambda 等第三方工具来曲线救国。

- Android 选择 Java 作为开发语言，在享受到 Java 已有大量开发者及开发资源的同时，也陷入了与 Oracle 旷日持久的诉讼大战中, 虽然目前情况是 Google 大胜，但 Google 想要绕过 Oracle 直接对 Java 进行改进仍然是不可能的。

- Google 在将编译和运行环境替换为 OpenJDK 后，必然需要一套全新的编译工具来支持其针对 Android 平台做出各种优化。

### 如何在项目中使用 JACK

构建环境为 Android Studio 2.1 Preview 5+Android Gradle Plugin 2.1.0+Build tools 24-rc3

默认的工程模板中没有启用 JACK，需要手动开启：

```
compileSdkVersion 'Android-N'
buildToolsVersion "24-rc3"
defaultConfig {
applicationId "com.example.JACKdemo"
minSdkVersion 24 
targetSdkVersion 'Android-N'
JACKOptions {
enabled true
   }
```

加入以上代码。如果需要对 Java 1.8语法的支持，还需要加入：

```
compileOptions {
sourceCompatibility JavaVersion.VERSION_1_8
targetCompatibility JavaVersion.VERSION_1_8
 }
```

如果还需要使用 Default Method和Repeatable Annotations，将minSdkVerson 设为24。

JACK 中默认带有混淆和无用资源裁剪，所以需要关掉默认构建文件中的相关选项，如 minifyEnabled。另外，JACK 默认并不会生成 Java 字节码文件，所以依赖于 Java 字节码文件的插件，如 FindBugs、Lint、Jacoco、Retrolambda 等需要关闭, 而用于提高调试效率的 Instant Run 也同样无法使用。

命令行中执行 gradle-q tasks， 如果列表中存在 compileDebugJavaWithJack 等相关 task，说明 JACK 编译已经启用了。

### JACK 的源代码

目前为止找不到太多关于 JACK 的资料，关于 JACK 内部的实现只能通过JACK的源代码进行简单的解析。JACK 的源代码库在 https://Android.googlesource.com/toolchain/JACK，通过对比Log、ub-jack 应该为开发主分支（如图3所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d39cdbf640.png" alt="图3 JACK的源代码工程目录" title="图3 JACK的源代码工程目录" />

图3 JACK 的源代码工程目录

从图3中可以看到，除了编译所需的库并移植 jarjar 和 dx 等已有工具之外，JACK 新的代码里已经添加了对覆盖率工具 Jacoco 的支持（Jacoco&jack-coverage）。

比较核心的有三个工程，如表1所示：

表1 UACK 核心工程

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d39d8e9bfa.jpg" alt="表1 UACK核心工程" title="表1 UACK核心工程" />

JACK 工程下的 frontend 和 backend 包分别是编译器前端和后端的组件代，JACK 的前端处理使用了定制的 ecj，将 Java 源码处理为 IR jayce, 而后端将 jayce 转化为 samli，并负责了混淆，Multidex 处理以及最终的 dex 输出。对于替代了 Java 字节码的全新的 IR jayce，目前并未找到相关的文档描述。

### 主要新特性及实测

#### 对 Java 8新语法的支持

JACK 所支持的 Java8 新特性并非全量，表2为部分新语法的测试结果。

表2 新语法支持

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d39e5dfbb2.png" alt="表2 新语法支持" title="表2 新语法支持" />

开发者最为关心的新特性应该是对lambda表达式的支持。通过反编译生成的代码可以发现，Retrolambda 与 JACK 均为通过将 lambda 表达式转化为实现了接口的匿名内部类的方式来实现。不同点在于 JACK 生成的代码中直接生成了匿名内部类，而 Retrolamdba 生成的类为单例模式，但这一点区别对程序运行没有什么可见影响。以下面 lambda 的实现为例：

```
static interface StringLoader{
     public String load();
 }
  
private String loadString(StringLoader l)
 {
     return l.load();
 }
  
 loadString(()->{return "default";});
```

首先是 Retrolambda 方式：

```
//Retrolambda所生成的lambda表达式构造函数。
 method static constructor <clinit>()V
     .locals 1
  
     new-instance v0, Lcom/example/JACKdemo/MainActivity$$Lambda$1;
  
     invoke-direct {v0}, Lcom/example/JACKdemo/MainActivity$$Lambda$1;-><init>()V
  
     sput-object v0, Lcom/example/JACKdemo/MainActivity$$Lambda$1;->instance:Lcom/example/JACKdemo/MainActivity$$Lambda$1;
  
     return-void
 .end method</init></clinit>
```

对比之下只有实现方式上的区别：    

```
//JACK生成的lambda表达式实现
 # direct methods
     .method public synthetic constructor <init>()V
     .locals 0
  
     invoke-direct {p0}, LJava/lang/Object;-><init>()V
  
     return-void
 .end method
  
 # virtual methods
 .method public string()LJava/lang/String;
    .locals 1
  
     .prologue
     .line 32
     const-string/jumbo v0, "Lambda"
  
      return-object v0
 .end method</init></init>
```

而JACK 对 Defalut Methods 的支持则有些奇怪，官方文档声明其运行需要 API level 23+，实际编译时如果 MinsdkVersion 低于24，Gradle 插件会直接提示错误：Default method not sup-ported in Android API level less than 24，而 API level 24的真机尚未上市，造成该特性事实上的无法使用。但是根据反编译的结果:

```
# virtual methods
 .method public init()LJava/lang/String;
     .locals 1
  
     .prologue
     .line 8
     const-string/jumbo v0, "default"
  
     return-object v0
 .end method
```

JACK 的实现基本与 Retrolambda 一致，并没有使用到新增的 opcode，通过修改所编译出的 APK 包测试也可以在低版本上正常使用，目前还不知道 Google 是否因计划后续修改实现方式才做出该限制。

#### 更快的编译速度

为了提高编译速度，JACK 提供了 pre-dexing 与增量编译，通过将 Libray 中的代码直接编译为 dex 格式以及保留源文件未修改的编译结果来减少编译中的耗时, 但目前 Gradle 插件中增量编译功能尚未开启，只能通过命令行中增加-D jack.incremental=true 来使用。实际上，在现有的 Grale 插件+Java+Proguard 组成的编译工具链中也提供了相同的功能，

Predex 在 Android Gradle tools 中被默认设置为“关闭”状态，增量编译 Android Gradle tools 2.1.0-rc1 之后默认设置为“打开”状态。两者可以通过以下代码手动切换：

```
“android {
 dexOptions {
     incremental true
     preDexLibraries true
 }
 }
```

目前 pre-dexing 和增量编译均与混淆冲突，只能用于 debug 状态下减少构建时间，在带有混淆的构建过程中如果开启，额外的dx过程反而会增加编译时间。表3是一个约9万方法数，依赖23个外部 Library 的应用在 JACK&Java+dx 开启了 pre-dex 和增量编译下的数次 debug 编译结果比较。

表3 不同工程编译比较

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d39fd1af21.jpg" alt="表3 不同工程编译比较" title="表3 不同工程编译比较" />

JACK 首次编译消耗了高达16分钟，远远超过了 Java+dx，其中 JILL 相关 task 所消耗的时间约为3分钟。编译过程中多次吃光了我的8 G 内存后崩溃，这使得它的实用性大打折扣。另外，启动时会有一行小小的 Log：Incremental Java compilation is an incubating feature，由此来看目前增加编译速度这一项应该还只是半成品状态。

#### 内嵌的混淆，优化与 multidex

混淆与资源优化的 JACK 实现位于 jack/com/Android/jack/shrob/下，同样并不全量支持 Proguard 的语法，具体定义可以在 proguard/proguard.g 或官方文档中找到。限于篇幅不一一列出。

但是在不支持的语法中有三个比较重要：

- dontnote

- dontwarn

目前的某些库需要添加这两项，否则 Gradle 插件在编译时会报错停止。

- assumenosideeffects 

通常用在 Release 删除日志等不需要包含在发布版本里的代码：

```
-assumenosideeffects class Android.util.Log {
     public static boolean isLoggable(Java.lang.String, int);
     public static int v(...);
 }
```

类似的功能无法继续在 JACK 中使用。

Multidex 的支持则包含在 backend/dex/multidex 包下，gradle 插件中，当 Api level < 21 时与目前的实现完全一致。

而对于 Api level >=21 (5.0+)时会使用 ART 原生的 Multidex 实现。

#### Gradle Plugin 的支持

从 com/android/build/gradle/tasks/JackTask 中可以看到 JACK 在编译时会接收的可设置 Gradle 脚本参数，如表4所示。

表4 Gradle 脚本参数

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d3a06d8161.jpg" alt="表4 Gradle脚本参数" title="表4 Gradle脚本参数" />

所有相关参数的使用方式与传统的 Java+dx 的方式完全一致。而 jackOptinos 所直接提供的参数只有2个：enabled 与 jackInProcess。而 JACK 自身支持的68个参数，插件中并无直接设置的方法。

### 从现有构建系统上迁移

#### 美团的代码静态检查系统 archon 的 JACK 部署测试

archon 是由一系列插件、脚本、工具组成的静态检测工具链，也是美团内部在使用的开发过程规范化工具。archon 中的部分功能，如 findebugs、PMD 等都是基于标准的 Java 字节码处理，但是 JACK 根本不会生成 Java 字节码文件，所有相关工具无法再直接使用，Jayce 格式资料不多，基于新的 IL 的处理工具尚未发现。

不过在对 JACK 工具链进行分析后，我们根据工具的不同类型找到了下面一些解决方式。

- 代码分析类工具：findebugs、PMD 等必须依赖字节码的工具，由于只需要编译到 Java 字节码阶段即可，在自动构建系统上保留了一个专门用于代码检测的工程，仍然使用 javac 进行编译，构建过程只执行到 compileJava，然后运行扫描。

- 构建过程 AOP 工具：我们自己构建了部分 AOP 工具，比如消除冗余 field，从字节码下沉到了 Smali 层实现得以保留，但对于 AspectJ 这种比较复杂的存在，暂时未有直接的解决方案。

- Kotlin：编译结果为标准的字节码文件，除使用 JILL 之外别无他法，Kotlin 官方已经在解决与 JACK 的兼容问题。

- 通用解决方案：使用 Java 编译生成字节码，处理之后通过 JILL 转换，这种叠床架屋的解决方式大部分情况下并不实用。

- 等待 JACK 官方支持，比如 Jacoco 在最新的 JACK 代码中已经添加。但就目前的实现来说，即使日后添加了对 findbugs 等类似工具的支持，也需要相当长的时间才能达到现有工具的水平。

- assumenosideeffects 失效的问题，对于小型项目来说，可以通过脚本直接清理源码或通过自行判断 debug 状态实现，但对于包含大量第三方库的大型工程，目前没有太好的解决方式。

### 当前存在的缺陷和限制

- Android N 目前只是早期预览版，带有不少 Bug，如某些混淆过的库在进行转换时报错，只能尝试更新源码自行编译一个版本替换或等待官方修复。

- Android Gradle  Build tools 里也带有一些 Bug，比如 JILL 超长的 build 时间，以及经常构建出来无 dex 的 APK 或消耗掉全部内存后假死（写这篇文章时，花费在处理各种 Bug 的时间远远的超过了真正的测试时间）。

- 目前 Gradle Jack Plugin 对于 JACK 的直接选项只有2个：enabled 是否启用；jackInProcess 是否工作在 Gradle 同一进程，jack-server 的参数需要通过 $HOME/.jack 来配置，而 JACK 自身共68个参数只有少数可以配置，使用起来相当不便。

- 现有的 Android 库全部使用 jar 或 aar 格式发布，使用的是 Java 字节码，编译中首先会由 JILL 转化为.jack 才能由 JACK 进行处理。对于比较大的项目，比如美团的主 App，使用 JACK 后一次完整的编译时间飙升至无法接受的43分钟。

### 结束语

总体来说，虽然 Google 对 JACK&JILL 寄予厚望，并计划将其设为默认编译工具链，但目前 JACK 也还只是一个预览版本，即使不考虑 Bug 造成的影响，全面替换后工具链上下游功能的缺失也会大大影响开发效率。但从长远来看，Google 统一工具链所带来的开发环境改善和针对 Android 的特有优化还是值得期待的。
