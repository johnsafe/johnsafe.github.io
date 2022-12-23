## Android 平台的崩溃捕获机制及实现

文/贾志凯

>Android 系统碎片化造成应用程序崩溃严重，在模拟器上运行良好的程序安装到某款手机上说不定就会出现崩溃的现象。而且，往往都是程序发布之后在用户端出现了崩溃现象。所以，如何及时捕获并收集 Android 平台的崩溃就显得愈发重要。目前，市面上已经有第三方 SDK 可以帮助开发者完成这一功能，本文将跟大家分享一下这些崩溃分析 SDK 的实现原理。

常见的 Android 崩溃有两类，一类是 Java Exception 异常，一类是 Native Signal 异常。我们将围绕这两类异常进行。对于很多基于 Unity、Cocos 平台的游戏，还会有 C#、JavaScript、Lua 等的异常，这里不做讨论。

### Java 代码的崩溃机制及实现

Android 应用程序的开发是基于 Java 语言的，所以首先来分析第一类 Android 崩溃 Java Exception。

#### Exception 的分类及捕获

Java 的异常可以分为两类：Checked Exception 和 UnChecked Exception。所有 RuntimeException 类及其子类的实例被称为 Runtime 异常，即 UnChecked Exception，不是 RuntimeException 类及其子类的异常实例则被称为 Checked Exception。

Checked 异常又称为编译时异常，即在编译阶段被处理的异常。编译器会强制程序处理所有的 Checked 异常，也就是用 try...catch 显式的捕获并处理，因为 Java 认为这类异常都是可以被处理（修复）的。在 Java API 文档中，方法说明时，都会添加是否 throw 某个 exception，这个 exception 就是 Checked 异常。如果没有 try...catch 这个异常，则编译出错，错误提示类似于“Unhandled exception type xxxxx”。

该类异常捕获的流程是：

- 执行 try 块中的代码出现异常，系统会自动生成一个异常对象，并将该异常对象提交给 Java 运行环境，这个就是异常抛出（throw）阶段；

- 当 Java 运行环境收到异常对象时，会寻找最近的能够处理该异常对象的 catch 块，找到之后把该异常对象交给 catch 块处理，这个就是异常捕获（catch）阶段。

Checked 异常一般是不引起 Android App Crash 的，注意是“一般”，这里之所以介绍，有两个原因：

- 形成系统的了解，更好地对比理解 UnChecked Exception；

- 对于一些 Checked Exception，虽然我们在程序里面已经捕获并处理了，但是如果能同时将该异常收集并发送到后台，将有助于提升 App 的健壮性。比如修改代码逻辑回避该异常，或者捕获后采用更好的方法去处理该异常。至于应该收集哪些 Checked Exception，则取决于 App 的业务逻辑和开发者的经验了。

UnChecked 异常又称为运行时异常，即 Runtime-Exception，最常见的莫过于 NullPointerException。UnChecked 异常发生时，由于没有相应的 try...catch 处理该异常对象，所以 Java 运行环境将会终止，程序将退出，也就是我们所说的 Crash。当然，你可能会说，那我们把这些异常也 try...catch 住不就行了。理论上确实是可以的，但有两点会导致这种方案不可行：

- 无法将所有的代码都加上 try...catch，这样对代码的效率和可读性将是毁灭性的；

- UnChecked 异常通常都是较为严重的异常，或者说已经破坏了运行环境的。比如内存地址，即使我们 try...catch 住了，也不能明确知道如何处理该异常，才能保证程序接下来的运行是正确的。

没有 try...catch 住的异常，即 Uncaught 异常，都会导致应用程序崩溃。那么面对崩溃，我们是否可以做些什么呢？比如程序退出前，弹出个性化对话框，而不是默认的强制关闭对话框，或者弹出一个提示框安慰一下用户，甚至重启应用程序等。

其实 Java 提供了一个接口给我们，可以完成这些，这就是UncaughtExceptionHandler，该接口含有一个纯虚函数：public abstract void uncaughtException (Thread thread, Throwable ex)。

Uncaught 异常发生时会终止线程，此时，系统便会通知 UncaughtExceptionHandler，告诉它被终止的线程以及对应的异常，然后便会调用 uncaughtException 函数。如果该 handler 没有被显式设置，则会调用对应线程组的默认 handler。如果我们要捕获该异常，必须实现我们自己的 handler，并通过函数 public static void setDefaultUncaughtExceptionHandler (Thread.UncaughtExceptionHandler handler)进行设置。实现自定义的 handler，只需要继承 UncaughtExceptionHandler 该接口，并实现 uncaughtException 方法即可。

```
static class MyCrashHandler implements UncaughtExceptionHandler{  
    @Override  
    public void uncaughtException(Thread thread, final Throwable throwable) {  
        // Deal this exception
    }
}
```

在任何线程中，都可以通过 setDefaultUncaughtExceptionHandler 来设置 handler，但在 Android 应用程序中，全局的 Application 和 Activity、Service 都同属于 UI 主线程，线程名称默认为“main”。所以，在 Application 中应该为 UI 主线程添加 UncaughtExceptionHandler，这样整个程序中的 Activity、Service 中出现的 UncaughtException 事件都可以被处理。

如果多次调用 setDefaultUncaughtExceptionHandler 设置 handler，以最后一次为准。这也就是为什么多个抓崩溃的 SDK 同时使用，总会有一些 SDK 工作不正常。某些情况下，用户会既想利用第三方 SDK 收集崩溃，又想根据崩溃类型做出业务相关的处理。此时有两种方案：

- SDK 提供了一个接口，允许用户传递自己 handler 给 SDK。当执行 uncaughtException 的时候，SDK 只负责采集崩溃信息，之后便调用用户的 handler 来处理崩溃了。这儿其实类似于多了一层 setDefaultUncaughtExceptionHandler，只是不能和标准库用同样的名称，因为会覆盖。

- 用户先通过 setDefaultUncaughtExceptionHandler 设置自己的 handler，然后 SDK 再次通过 setDefaultUncaughtExceptionHandler 设自 handler 前，先保存之前的 handler，在 SDK handler 逻辑处理完成之后，再调用之前的 handler 处理该异常。

```
static class MyCrashHandler implements UncaughtExceptionHandler {
    private UncaughtExceptionHandler originalHandler;
    private MyCrashHandler(Context context) {
        originalHandler = Thread.getDefaultUncaughtExceptionHandler();
    }
    @Override
    public void uncaughtException(Thread thread, final Throwable throwable) {
        // Deal this exception
        if (originalHandler != null) {
            originalHandler.uncaughtException(thread, throwable);
        }
    }
}
```

#### 获取 Exception 崩溃堆栈

捕获 Exception 之后，我们还需要知道崩溃堆栈的信息，这样有助于我们分析崩溃的原因，查找代码的 Bug。异常对象的 printStackTrace 方法用于打印异常的堆栈信息，根据 printStackTrace 方法的输出结果，我们可以找到异常的源头，并跟踪到异常一路触发的过程。

```
public static String getStackTraceInfo(final Throwable throwable) {
    String trace = "";
    try {
        Writer writer = new StringWriter();
        PrintWriter pw = new PrintWriter(writer);
        throwable.printStackTrace(pw);
        trace = writer.toString();
        pw.close();
    } catch (Exception e) {
        return "";
    }
    return trace;
}
```

### Native 代码的崩溃机制及实现

Android 平台除了使用 Java 语言开发以外，还提供了对 C/C++ 的支持。对于一些高 CPU 消耗的应用程序，Java 语言很难满足对性能的要求，这时就需要使用 C/C++ 进行开发，比如游戏引擎、信号处理等。但是 Native 代码只能开发动态链接库（so），然后 Java 通过 JNI 来调用 so 库。

#### Native 崩溃分析与捕获

C++ 也可以通过 try...catch 去处理一些异常，但如果出现了 Uncaught 异常，so 库就会引起崩溃。此时肯定无法通过 Java 的 Uncaught-ExceptionHandler 来处理，那么我们应该如何捕获 Native 代码的崩溃呢？熟悉 Linux 开发的人都知道，so 库一般通过 gcc/g++ 编译，崩溃时会产生信号异常。Android 底层是 Linux 系统，所以 so 库崩溃时也会产生信号异常。那么如果我们能够捕获信号异常，就相当于捕获了 Android Native 崩溃。

信号其实是一种软件层面的中断机制，当程序出现错误，比如除零、非法内存访问时，便会产生信号事件。那么进程如何获知并响应该事件呢？Linux 的进程是由内核管理的，内核会接收信号，并将其放入到相应的进程信号队列里面。当进程由于系统调用、中断或异常而进入内核态以后，从内核态回到用户态之前会检测信号队列，并查找到相应的信号处理函数。内核会为进程分配默认的信号处理函数，如果你想要对某个信号进行特殊处理，则需要注册相应的信号处理函数（如图1所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201601/568b3052c256d.png" alt="图1  信号检测与处理流程（原图出自《Linux内核源代码情景分析》）" title="图1  信号检测与处理流程（原图出自《Linux内核源代码情景分析》）" />

图1  信号检测与处理流程（原图出自《Linux内核源代码情景分析》）

进程对信号异常的响应可以归结为以下几类： 

- 忽略信号：对信号不做任何处理，除了 SIGKILL及SIGSTOP 以外（超级用户杀掉进程时产生），其他都可以忽略；

- 捕获信号：注册信号处理函数，当信号发生时，执行相应的处理函数；

- 默认处理：执行内核分配的默认信号处理函数，大多数我们遇到的信号异常，默认处理是终止程序并生成 core 文件。

对 Native 代码的崩溃，可以通过调用 sigaction()注册信号处理函数来完成。

```
#include <signal.h> 
int sigaction(int signum,const struct sigaction *act,struct sigaction *oldact));</signal.h>
```

- signum：代表信号编码，可以是除 SIGKILL 及 SIGSTOP 外的任何一个特定有效的信号，如果为这两个信号定义自己的处理函数，将导致信号安装错误。

- act：指向结构体 sigaction 的一个实例的指针，该实例指定了对特定信号的处理，如果设置为空，进程会执行默认处理。

- oldact：和参数 act 类似，只不过保存的是原来对相应信号的处理，也可设置为 NULL。

sigaction 函数用于改变进程接收到特定信号后的行为。如果把第二、第三个参数都设为 NULL，那么该函数可用于检查信号的有效性。

结构体 sigaction 包含了对特定信号的处理、信号所传递的信息、信号处理函数执行过程中应屏蔽掉哪些函数等等。

```
struct sigaction {
    void     (*sa_handler)(int);
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask;
    int        sa_flags;
    void     (*sa_restorer)(void);
}
```

在一些体系上，sa_handler 和 sa_sigaction 共用一个联合体（union），所以不要同时指定两个字段的值。

- sa_handler： 指定对 signum 信号的处理函数，可以是 SIG_DFL 默认行为，SIG_IGN 忽略接送到的信号，或者一个信号处理函数指针。这个函数只有信号编码一个参数。

- sa_sigaction： 当 sa_flags 中存在 SA_SIGINFO 标志时，sa_sigaction 将作为 signum 信号的处理函数。

- sa_mask：指定信号处理函数执行的过程中应被阻塞的信号。

- sa_flags：指定一系列用于修改信号处理过程行为的标志，由0个或多个标志通过 or 运算组合而成，比如 SA_RESETHAND，SA_ONSTACK | SA_SIGINFO。

- sa_restorer： 已经废弃，不再使用。

各参数的详细使用说明，请参考相关资料。

基于上面的分析，下面给出 Native 代码崩溃（即信号异常）捕获的代码片段，让大家有一个更直观的认识。

```
const int handledSignals[] = {
    SIGSEGV, SIGABRT, SIGFPE, SIGILL, SIGBUS
};
const int handledSignalsNum = sizeof(handledSignals) / sizeof(handledSignals[0]);
struct sigaction old_handlers[handledSignalsNum];
 
int nativeCrashHandler_onLoad(JNIEnv *env) {
    struct sigaction handler;
    memset(&handler, 0, sizeof(sigaction));
    handler.sa_sigaction = my_sigaction;
    handler.sa_flags = SA_RESETHAND;
 
    for (int i = 0; i < handledSignalsNum; ++i) {
        sigaction(handledSignals[i], &handler, &old_handlers[i]);
    }
 
    return 1;
}
```

当 Android 应用程序加载 so 库的时候，调用 nativeCrashHandler_onLoad 会为 SIGSEGV、SIGABRT、SIGFPE、SIGILL、SIGBUS 通过 sigaction 注册信号处理函数 my_sigaction。当发生 Native 崩溃并且发生前面几个信号异常时，就会调用 my_sigaction 完成信号处理。

```
notifyNativeCrash  = (*env)->GetMethodID(env, cls,  "notifyNativeCrash", "()V");
void my_sigaction(int signal, siginfo_t *info, void *reserved) {
    // Here catch the native crash
}
```

#### 获取 Native 崩溃堆栈

Android 没有提供像 throwable.printStackTrace 一样的接口去获取 Native 崩溃后堆栈信息，所以我们需要自己想办法实现。这里有两种思路可以考虑。

- 利用 LogCat 日志

在本地调试代码时，我们经常通过查看 LogCat 日志来分析解决问题。对于发布的应用，在代码中执行命令“logcat -d -v threadtime”也能达到同样的效果，只不过是获取到了用户手机的 logcat。当 Native 崩溃时，Android 系统同样会输出崩溃堆栈到 LogCat，那么拿到了 LogCat 信息也就拿到了 Native 的崩溃堆栈。

```
Process process = Runtime.getRuntime().exec(new String[]{"logcat","-d","-v","threadtime"});
String logTxt = getSysLogInfo(process.getInputStream());
```

在 my_sigaction 捕获到异常信号后，通知 Java 层代码，在 Java 层启动新的进程，并在新的进程中完成上面的操作。这里注意一定要在新的进程中完成，因为原有的进程马上就会结束。

网络上有一些对应这种思路的代码，但是在很多手机上都无法获得 Native 的崩溃堆栈。原因是对崩溃堆栈产生了破坏，使得相关信息并没有输出到 logcat 中。研究一下 Android backtrace 的底层实现以及 Google Breakpad 的源码，会帮助你解决这个问题。

- Google Breakpad

Linux 提供了 Core Dump 机制，即操作系统会把程序崩溃时的内存内容 dump 出来，写入一个叫做 core 的文件里面。Google Breakpad 作为跨平台的崩溃转储和分析模块（支持 Windows、OS X、Linux、iOS 和 Android 等），便是通过类似的 MiniDump 机制来获取崩溃堆栈的。

通过 Google Breakpad 捕获信号异常，并将堆栈信息写入你指定的本地 MiniDump 文件中。下次启动应用程序的时候，便可以读取该 MiniDump 文件进行相应的操作，比如上传到后台服务器。当然，也可以修改 Google Breakpad 的源码，不写 MiniDump 文件，而是通过 dumpCallback 直接获得堆栈信息，并将相关信息通知到 Java 层代码，做相应的处理。

Google Breakpad 是权威的捕获 Native 崩溃的方法，相关的使用方法可以查看官网文档。由于它跨平台，代码体量较大，所以建议大家裁剪源码，只保留 Android 相关的功能，保持自己 APK 的小巧。