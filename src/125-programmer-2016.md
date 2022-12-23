## 深入浅出 Android 打包

文/芈峮

>Android 市场的渠道分散已不是什么新鲜事，但如何高效打包仍是令许多开发者头疼的问题。本篇文章着重介绍了目前最新的三种打包方案，并且从安全方面对这三种方案进行点评，相信会给开发者带来新的助力。

一般需求的打包，一条行命令就出来了。复杂一些的话，也就是一个简单的开源工具，或是一段小配置代码就搞定了。既然如此，为什么我还要来写 Android 打包相关内容？主要有以下两个方面的原因：

- Android 市场开放，渠道越来越多，几千个渠道已经是常态。所以，需要一个高效且安全的打包方式，来支持成千上万的渠道打包重任。

- 现在 Android 应用程序都会使用 ProGuard 或者 DexGuard 来进行代码混淆，在引入代码混淆的前提下，需要针对某一个版本尽量只保留一份符号表（Android 是 mapping 文件）。多份不同的符号表对应到一个版本的程序，会给 Crash 修正等后期问题的追查造成很大困难。

所以，当 Android 开发团队有一定规模时，需要一种优雅并且高效安全的打包方式来衔接从代码到应用程序包的这个过程。

### 历史回顾

大概在3-4年前，国内的移动开发者都会使用友盟来做 Crash 收集和一些用户的行为数据收集。所以，友盟定义渠道的方式基本定义了数据收集工具的渠道定义方式。友盟默认的渠道直接从 AndroidManifest 文件中的 meta－data 里面读取。如以下代码所示：

```
'' <meta-data ''="" android:name="UMENG_APPKEY" android:value="4ee5c714ee5c714ee5c71">
'' </meta-data>
'' <meta-data ''="" android:name="UMENG_CHANNEL" android:value="wandoujia">
'' </meta-data>
```

在示例中，UMENG\_APPKEY 是账户的唯一标示，UMENG\_CHANNEL 的内容为 wandoujia，即表示这时渠道是 wandoujia。这里再多说一句渠道的概念，一般的渠道可能会表示具体是哪个应用市场。比如上传豌豆荚和上传360手机助手的渠道标示不一样，当然，如果有一些线下推广的话，也需要区分渠道。随着数据统计越来越细，渠道也会更加的层出不穷。现在一个日活在百万的应用，Android 的渠道在几千个是非常正常的现象。

在 AndroidManifest 定义渠道的年代，多渠道打包无非以下两种方案：

- 方案一：完全的重新编译，即在代码重新编译打包之前，在 AndroidManifest 中修改渠道标示；

- 方案二：通过 ApkTool 进行解包，然后修改 AndroidManifest 中修改渠道标示，最后再通过 ApkTool 进行打包、签名。

现在看来，这两种方式效率都不高。方案一基本上毫无效率可言，只能支持20个以内的渠道规模。方案二效率高一些，如果再引入多线程驱动来进行打包，支持200以内的渠道打包毫无压力。但是随着 Android 的版本升级，代码混淆和加固工具的引入，这种方案已经不能正常运行了。所以，需要一种兼容性好并且高效的多渠道打包方式。

### 三种高效的多渠道打包方式

Android项目本身是开源的，大家都可以去研究任何实现，基于这个特性，一种叫做“Android黑科技”的学科迅速诞生。以下介绍的多渠道打包方式都应该属于其范畴。

#### 添加 comments 多渠道打包

首先解释什么是 comments （注释或评论）的打包方式？

Android 应用使用的 APK 文件就是一个带签名信息的 zip 文件，根据 zip 文件格式规范（请参见：https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT），每个文件的最后都必须有一个叫 Central Directory Record（https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html）的部分，这个 CDR 的最后部分叫“End of Central Directory Record”，这一部分包含一些元数据，它的末尾是 zip 文件的注释。注释包含 Comment Length 和 File Comment 两个字段。前者表示注释的长度，后者是注释的内容，正确修改这一部分不会对 zip 文件造成破坏，利用这个字段，我们可以添加一些自定义数据。

简单来说就是：我们利用的是 zip 文件“可以添加 comment （摘要）”的数据结构特点，在文件的末尾写入任意数据，而不用重新解压 zip 文件（apk 文件就是 zip 文件格式）；所以该工具不需要对 APK 文件解压缩和重新签名即可完成多渠道自动打包，可谓是高效、速度快，无兼容问题。

这种高效的方式是奇虎360的一位工程师公布的，他已经把相关工具的代码开放在 GitHub 上面了，地址为https://github.com/seven456/MultiChannelPackageTool。工具本身为命令行形式，一条命令即可生成一个多渠道包，并且非常高效。作者本人公布的性能数据是：5 M 的 APK，1秒种能打300个。

就是因为速度非常快，现在一些厂商用这种方案来做安装包的动态生成。在用户下载之前渠道是没有添加进去的，用户选择下载以后通过渠道来源判断渠道标示，并且写入。过程瞬间完成，用户下载时毫无感知。

当然，利用工具把渠道写入应用程序包内，还应该提供一种在 APK 内部读取渠道的方式。读取时方法也非常简单，首先通过反射方法找到 APK 在手机中的位置，然后通过读 zip comment 的方式获取渠道内容。读取 comment 和找 APK 位置的代码都能在这个急速添加渠道的工具中找到，在这里不再详细介绍。需要注意的是，在 Android 中使用 Java 反射的性能非常差，需要做一些优化处理。

如果你觉得这种方式的安全比较差，也可以自己重新实现一个小工具，并且对 comment 的内容加密。当然，加密使用的密钥需要在 APK 代码中体现，这种加密方案也只是提高了一点破译的门槛而已。

#### 美团的 Android 多渠道打包

美团点评技术团队（原美团技术团队）有一个公开的博客，每篇文章都在领域内有一定的影响。其中有一篇就是讨论“如何高效地进行 Android 多渠道打包？”，可参见：http://tech.meituan.com/mt-apk-packaging.html。

文章中首先提到了ApkTool的多渠道打包方式，然后讲到美团的业务发展已经有900多个渠道，ApkTool的方式已经不能支持这种规模的多渠道打包。所以，美团点评技术团队开始探索一种更加高效的多渠道打包方式。

美团的做法是把一个 Android 应用包当作 zip 文件包进行解压，然后发现在签名生成的目录下添加一个空文件不需要重新签名。利用这个机制，该文件的文件名就是渠道名。这种方式不需要重新签名等步骤，非常高效。

下面我们来详细讲解一下这个过程。首先把一个已经签名好的 Android 应用程序包解压缩，签名文件的目录为：META-INF，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb3b2b0ed22.jpg" alt="图1  美团Android多渠道打包压缩文件目录" title="图1  美团Android多渠道打包压缩文件目录" />

图1  美团 Android 多渠道打包压缩文件目录

图中有两个框选的部分，分别为：assets 目录和 META－INF 目录。assets 等下文介绍豌豆荚高效打包方式的时候再说。我们这里先来看看 META－INF 目录。在 META－INF 目录下的三个文件，就是一个 Android 应用程序包被正确签名之后的生成文件，当这个程序包中的文件被修改以后，验证信息就需要重新生成，否则验证的签名信息就不正确。这套签名机制也是从 Java 的 jar 包方式继承过来的。

美团多渠道打包方式，正是利用签名的验证方式，巧妙地放入了一个空文件，利用文件名来表示渠道方式的完成。如果放入了一个非空的文件，这时签名验证的机制就会被破坏。美团的具体做法是通过一段简单的 Python 脚本来完成。代码如下：

```
'' import zipfile
'' zipped = zipfile.ZipFile(your_apk, 'a', zipfile.ZIP_DEFLATED) 
'' empty_channel_file = "META-INF/mtchannel_{channel}".format(channel=your_channel)
'' zipped.write(your_empty_file, empty_channel_file)
```

通过运行 Python 脚本，Android 应用程序包中便会多出一个文件，如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb3b59d0e22.jpg" alt="图2  通过运行Python脚本，Android应用程序包中多出来的一个文件" title="图2  通过运行Python脚本，Android应用程序包中多出来的一个文件" />

图2  通过运行 Python 脚本，Android 应用程序包中多出来的一个文件

渠道添加成功后，可以参照美团点评技术团队的博客，在 Android 程序中添加读取渠道的代码，即可拥有高效的多渠道打包方式。

当然，这个技术实现本身可以有更加优雅和高效的方式——添加渠道。原理不变，添加方式可以进行改变。添加文件可以使用 AAPT 这个工具。AAPT 本身是 Android 平台自带的一个命令行工具，可以实现从 Android 程序包中删除和添加任意文件。当然，如果添加文件破坏了签名验证信息，则是另外一回事了。关于 AAPT 的使用和其在多渠道签名打包时的用法，会在下文进行讲解。

如果现在想把美团的渠道删除，使用 AAPT 输入一条简单的命令行即可（能删除就能添加，添加部分还是在豌豆荚多渠道打包方案中介绍吧）：

```
'' aapt remove  meituan.apk META-INF/mtchannel_meituan
```

#### 豌豆荚 Android 多渠道打包

豌豆荚的多渠道打包方案和美团的所用的方式一样，都是添加一个文件，文件本身会带入渠道消息。但不同点在于它添加的是一个不为空的文件。之前已经提到过了，如果添加一个非空文件，就会破坏签名校验，需要重新签名。

从这个方案可以看出，豌豆荚多渠道打包和美团多渠道打包正好是利用了一个特性的两个方面。当然，如果需要重新签名，效率会差一些。在失去效率的同时，拥有了更加安全的渠道管理。下面具体讲解，豌豆荚多渠道打包管理方案。

首先，如果把渠道文件还放在 META－INF 目录下的话，重新签名时渠道文件会被删掉。所以，渠道文件需要重新找一个合理的地方存放。之前的文章也提到过，还有一个 assets 目录。assets 目录和 META－INF 目录一样，可以随便放入小文件。如果是空文件不需要重新签名

加入渠道时主要有以下步骤。

- 首先需要准备好渠道文件。在 Android 应用程序包的同级目录，新建一个 assets 目录，并且在该目录下放入一个 channel.txt 文件，文件内容是渠道信息，可以是 wandoujia 或360等。

- 使用 AAPT 加入渠道文件：

```
'' aapt add wandoujia.apk asserts/channel.txt
```

- 在渠道加入后，重新签名 APK 文件并且进行对齐。

重新签名相关的步骤就不再介绍了，如果感兴趣的话，推荐看一下 calabash-android 这个开源项目的 resign 相关部分实现，calabash 的重新签名实现是我见过最严谨的。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb402c5d02a.jpg" alt="图3  豌豆荚打包方案处理一个包需要大约4秒" title="图3  豌豆荚打包方案处理一个包需要大约4秒" />

图3  豌豆荚打包方案处理一个包需要大约4秒

说完了豌豆荚多渠道打包的实现部分以后，需要来谈一下效率。之前的两种多渠道打包方案效率都很高，处理一个包都能在1秒之内完成。但豌豆荚的多渠道打包方案由于必须重新签名，会慢一些（但也没有慢多少）。在屏蔽了一些关键信息之后，从图3中可以看到，一次处理大概需要4秒的时间。现在一台一般配置的计算机，开8个线程处理，速度完全可以接受。

### 从安全的角度看多渠道打包方案

三种打包方案都是高效的实现，当然也有各自的侧重点。以下，从安全的角度来分析。

对于 Android 应用包的安全问题来说，主要需要考虑渠道被恶意篡改的可能性和成本。如果一个渠道商，通过网络劫持和篡改渠道的组合方式来获取暴利，对于程序开发者来说可能会存在着巨大的经济损失。

在介绍篡改渠道之前，还是先简单介绍 AAPT 这个工具的一般使用方法。AAPT 是在 AndroidSDK 内的一个命令行工具，具体位置在 AndroidSDK 目录下的 build-tools/19.0.0或者其他版本下，AAPT 并不存在于每个发行版本中，所以一般使用的 AAPT 是19.0.0或19.0.1版本。

使用 AAPT 主要有以下4种场景：

- 查看 Android 开发包的基本信息，例如：包名、版本等。以微信的 Android 版本为例：

```
'' INPUT  apt dump badging weixin.apk
'' OUTPUT package: name='com.tencent.mm' versionCode='740' versionName='6.3.13.49_r4080b63'
'' uses-permission:'com.tencent.mm.plugin.permission.READ'
'' uses-permission:'com.tencent.mm.plugin.permission.WRITE'
'' uses-permission:'com.tencent.mm.permission.MM_MESSAGE'
''  uses-permission:'com.huawei.authentication.HW_ACCESS_AUTH_SERVICE'
'' sdkVersion:'15'
'' targetSdkVersion:'23'
'' uses-feature-not-required:'android.hardware.camera'
'' uses-feature-not-required:'android.hardware.camera.autofocus'
'' uses-feature-not-required:'android.hardware.bluetooth'
'' uses-feature-not-required:'android.hardware.location'
'' uses-feature-not-required:'android.hardware.location.gps'
'' uses-feature-not-required:'android.hardware.location.network'
'' uses-feature-not-required:'android.hardware.microphone'
'' uses-feature-not-required:'android.hardware.telephony'
'' uses-feature-not-required:'android.hardware.touchscreen'
'' uses-feature-not-required:'android.hardware.wifi'
'' uses-permission:'android.permission.ACCESS_NETWORK_STATE'
'' uses-permission:'android.permission.ACCESS_COARSE_LOCATION'
'' uses-permission:'android.permission.ACCESS_FINE_LOCATION'
'' uses-permission:'android.permission.CAMERA'
'' .....
```

从以上代码可以看到这个微信的版本信息，并且知道了微信的包名是：com.tencent.mm，还能看到微信这个应用使用到了哪些权限等信息（更多的信息还是交给读者自己仔细研究吧）。

- 使用 AAPT 查看 Android 开发包的目录结构如下：

```
'' INPUT  aapt list  weixin.apk
'' OUTPUT META-INF/MANIFEST.MF
''   META-INF/COM_TENC.SF
''   META-INF/COM_TENC.RSA
''   assets/jsapi/wxjs.js
''   assets/avatar/default_nor_avatar.png
''   assets/avatar/default_meishiapp.png
''   assets/avatar/default_hd_avatar.png
''   assets/ipcall_country_code.txt
''   assets/merged_features.xml
''   assets/address
''   .....
```

又看到了 META－INF 目录和 assets 目录。一般在 Android 的应用中 assets 都会放一些静态的资源。

- 使用 AAPT 添加一个文件：

```
'' INPUT aapt add wandoujia.apk  assets/channel.txt
```

使用 AAPT 删除一个文件

```
'' INPUT aapt remove wandoujia.apk assets/channel.txt
```

在掌握了以上简单的四个命令行后，修改美团的渠道就已经非常简单了。先通过 list 命令找到渠道文件，然后直接删除文件，最后再加入自己的恶意渠道。但豌豆荚的渠道也可以通过相同的方式来进行篡改，但篡改之后需要重新签名。在没有官方的签名文件时，是没有办法完全伪造一个相同的签名的。

添加 comments 多渠道打包方式中虽然渠道信息不是明文显示，但也存在着被篡改的可能性。如果自己进行一些简单的加密可以杜绝大多数的恶意篡改，但由于不需要重新签名，这种方法还是存在着被篡改的机率的。

具体选择哪一种多渠道打包方式，还是由业务决定的。豌豆荚作为一个应用商店，需要有自己的渠道，渠道的安全性会很不好。如果是一个普通的 App，一般都是在一些应用商店上发布新版本，应用商店本身会给开发者提供相对安全的环境。所以这也就是美团方案依然能够使用的关键。

### 结束语

本文重点介绍了3种高效的多渠道打包方式。这三种方式都不会产生二次编译，二次编译会对一个版本的程序产生多个版本的符号表，在后期的问题追查中会有很多障碍。利用结束语简单说说符号表的重要性吧。

当程序正确打包发布以后，可以通过各种免费、收费或自己开发的工具来收集 Crash 等堆栈信息，查看错误并进行改正。你可能会得到一段这样的异常堆栈：

```
'' ava.lang.RuntimeException: There is no view set with ViewPackage found in the log tree.
'' at o.λ.ˎ(:64)
'' at o.λ.ˊ(:25)
'' at o.к$ˋ.run(:233)
'' at android.os.Handler.handleCallback(Handler.java:733)
'' at android.os.Handler.dispatchMessage(Handler.java:95)
'' at android.os.Looper.loop(Looper.java:136)
'' at android.os.HandlerThread.run(HandlerThread.java:61)
```

然后如果没有一个符号表帮忙，你会陷入深深的沉思：这到底是什么问题？我的代码在哪里？如果有了正确的符号表，一条命令行就可以就可以把相关的异常堆栈信息翻译正确。符号表和编译过程有着紧密的耦合，所以在选择打包工具时，都应该选择不产生二次编译的工具。在以上提到的三款工具中，添加多渠道都是基于已有APK进行添加的，并不是基于源码来添加渠道。所以都完全满足条件。

考虑到文章的篇幅，有一些内容没有展开说，开发者如果遇到这方面的问题，欢迎与我交流。
