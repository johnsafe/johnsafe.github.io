## PHP 并发 I/O 编程之路

文/韩天峰

>并发 I/O 问题一直是服务器端编程中的技术难题，从最早的同步阻塞直接 Fork 进程，到 Worker 进程池/线程池，到现在的异步 I/O、协程。PHP 程序员因为有强大的 LAMP 框架，对这类底层方面的知识知之甚少，本文目的就是详细介绍 PHP 进行并发 I/O 编程的各种尝试，最后再介绍 Swoole 的使用，深入浅出全面解析并发 I/O 问题。

### 多进程/多线程同步阻塞

最早的服务器端程序都是通过多进程、多线程来解决并发 I/O 的问题。进程模型出现的最早，从 Unix 系统诞生就开始有了进程的概念。最早的服务器端程序一般都是接受一个客户端连接就创建一个进程，然后子进程进入循环同步阻塞地与客户端连接进行交互，收发处理数据。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d48fb6aec0.png" alt="图1  最初的进程模型示意图" title="图1  最初的进程模型示意图" />

图1  最初的进程模型示意图

多线程模式出现要晚一些，线程与进程相比更轻量，而且线程之间是共享内存堆栈的，所以不同的线程之间交互非常容易实现。比如聊天室这样的程序，客户端连接之间可以交互，聊天室中的玩家可以向任意其他人发消息。用多线程模式实现非常简单，线程中可以直接向某一个客户端连接发送数据。而多进程模式就要用到管道、消息队列、共享内存，统称进程间通信（IPC）复杂的技术才能实现。

代码实例：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4926f361b.png" alt="代码1" title="代码1" />

代码1

多进程/线程模型的流程是：

- 创建一个 Socket，绑定服务器端口（bind），监听端口（listen），在 PHP 中用 stream_socket_server 一个函数就能完成上面3个步骤，当然也可以使用更底层的 Sockets 扩展分别实现。

- 进入 While 循环，阻塞在 Accept 操作上，等待客户端连接进入。此时程序会进入随眠状态，直到有新的客户端发起 Connect 到服务器，操作系统会唤醒此进程。Accept 函数返回客户端连接的 Socket。

- 主进程在多进程模型下通过 Fork（php: pcntl_fork）创建子进程，多线程模型下使用 pthread_create（php: new Thread）创建子线程。下文如无特殊声明将使用进程同时表示进程/线程。

- 子进程创建成功后进入 While 循环，阻塞在 Recv（php: fread）调用上，等待客户端向服务器发送数据。收到数据后服务器程序进行处理然后使用 Send（php: fwrite）向客户端发送响应。长连接的服务会持续与客户端交互，而短连接服务一般收到响应就会 Close。

- 当客户端连接关闭时，子进程退出并销毁所有资源。主进程会回收掉此子进程。

这种模式最大的问题是，进程/线程创建和销毁的开销很大。所以上面的模式没办法应用于非常繁忙的服务器程序。对应的改进版解决了此问题，这就是经典的 Leader-Follower 模型。

代码实例：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4978611e1.png" alt="代码2" title="代码2" />

代码2

它的特点是程序启动后就会创建 N 个进程。每个子进程进入 Accept，等待新的连接进入。当客户端连接到服务器时，其中一个子进程会被唤醒，开始处理客户端请求，并且不再接受新的 TCP 连接。当此连接关闭时，子进程会释放，重新进入 Accept，参与处理新的连接。

这个模型的优势是完全可以复用进程，没有额外消耗，性能非常好。很多常见的服务器程序都是基于此模型的，比如 Apache、PHP-FPM。

多进程模型也有一些缺点。

- 这种模型严重依赖进程的数量解决并发问题，一个客户端连接就需要占用一个进程，工作进程的数量有多少，并发处理能力就有多少。操作系统可以创建的进程数量是有限的。

- 启动大量进程会带来额外的进程调度消耗。数百个进程时可能进程上下文切换调度消耗占 CPU 不到1%可以忽略不计，如果启动数千甚至数万个进程，消耗就会直线上升。调度消耗可能占到 CPU 的百分之几十甚至100%。

另外有一些场景多进程模型无法解决，比如即时聊天程序（IM），一台服务器要同时维持上万甚至几十万上百万的连接（经典的 C10K 问题），多进程模型就力不从心了。

还有一种场景也是多进程模型的软肋。通常Web服务器启动100个进程，如果一个请求消耗100 ms，100个进程可以提供1000 qps，这样的处理能力还是不错的。但是如果请求内要调用外网 HTTP 接口，像 QQ、微博登录，耗时会很长，一个请求需要10s。那一个进程1秒只能处理0.1个请求，100个进程只能达到10 qps，这样的处理能力就太差了。

有没有一种技术可以在一个进程内处理所有并发 I/O 呢？答案是有，这就是 I/O 复用技术。

### I/O 复用/事件循环/异步非阻塞

其实 I/O 复用的历史和多进程一样长，Linux 很早就提供了 select 系统调用，可以在一个进程内维持1024个连接。后来又加入了 poll 系统调用，poll 做了一些改进，解决了1024限制的问题，可以维持任意数量的连接。但 select/poll 还有一个问题就是，它需要循环检测连接是否有事件。这样问题就来了，如果服务器有100万个连接，在某一时间只有一个连接向服务器发送了数据，select/poll 需要做循环100万次，其中只有1次是命中的，剩下的99万9999次都是无效的，白白浪费了 CPU 资源。

直到 Linux 2.6 内核提供了新的 epoll 系统调用，可以维持无限数量的连接，而且无需轮询，这才真正解决了 C10K 问题。现在各种高并发异步 I/O 的服务器程序都是基于 epoll 实现的，比如 Nginx、Node.js、Erlang、Golang。像 Node.js 这样单进程、单线程的程序，都可以维持超过1百万 TCP 连接，全部归功于 epoll 技术。

I/O 复用异步非阻塞程序使用经典的 Reactor 模型，Reactor 顾名思义就是反应堆的意思，它本身不处理任何数据收发。只是可以监视一个 Socket 句柄的事件变化。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4ddc1e257.png" alt="图2  Reactor模型" title="图2  Reactor模型" />

图2  Reactor 模型

Reactor 有4个核心的操作：

- Add 添加 Socket 监听到 Reactor，可以是 Listen Socket 也可以使客户端 Socket，也可以是管道、eventfd、信号等。

- Set 修改事件监听，可以设置监听的类型，如可读、可写。可读很好理解，对于 Listen Socket 就是有新客户端连接到来了需要 Accept。对于客户端连接就是收到数据，需要 Recv。可写事件比较难理解一些。一个 Socket 是有缓存区的，如果要向客户端连接发送2M的数据，一次性是发不出去的，操作系统默认 TCP 缓存区只有256K。一次性只能发256 K，缓存区满了之后 Send 就会返回 EAGAIN 错误。这时候就要监听可写事件，在纯异步的编程中，必须去监听可写才能保证 Send 操作是完全非阻塞的。

- Del 从 Reactor 中移除，不再监听事件。

- Callback 就是事件发生后对应的处理逻辑，一般在 Add/Set 时制定。C 语言用函数指针实现，JS 可以用匿名函数，PHP 可以用匿名函数、对象方法数组、字符串函数名。

Reactor 只是一个事件发生器，实际对 socket 句柄的操作，如 Connect/Accept、Send/Recv、Close 是在 Callback 中完成的。具体编码可参考代码3（伪代码）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4e9c85dc1.png" alt="代码3" title="代码3" />

代码3

Reactor 模型还可以与多进程、多线程结合起来用，既实现异步非阻塞 I/O，又利用到多核。目前流行的异步服务器程序都是这样的方式：

- Nginx：多进程 Reactor；

- Nginx+Lua：多进程 Reactor+ 协程；

- Golang：单线程 Reactor+ 多线程协程；

- Swoole：多线程 Reactor+ 多进程 Worker。

#### 协程是什么

协程从底层技术角度看实际上还是异步 I/O Reactor 模型，应用层自行实现了任务调度，借助 Reactor 切换各个当前执行的用户态线程，但用户代码中完全感知不到 Reactor 的存在。

#### PHP 相关扩展

- Stream：PHP 内核提供的 Socket 封装；

- Sockets：对底层 Socket API 的封装；

- Libevent：对 Libevent 库的封装；

- Event：基于 Libevent 更高级的封装，提供了面向对象接口、定时器、信号处理的支持；

- Pcntl/Posix：多进程、信号、进程管理的支持；

- Pthread：多线程、线程管理、锁的支持；

- PHP还有共享内存、信号量、消息队列的相关扩展；

- PECL：PHP 的扩展库，包括系统底层、数据分析、算法、驱动、科学计算、图形等都有。

如果 PHP 标准库中没有找到，可以在 PECL 寻找想要的功能。

#### PHP 语言的优劣势

PHP的优点包括以下几点。

- 第一个是简单，PHP 比其他任何的语言都要简单，入门的话 PHP 真的是可以一周就够。有一本书叫做《21天深入学习 C++ 》，其实21天根本不可能学会，甚至可以说 C++ 没有3-5年不可能深入掌握。但是 PHP 绝对可以7天入门。所以 PHP 程序员的数量非常多，招聘比其他语言更容易。

- PHP 的功能非常强大，因为 PHP 官方的标准库和扩展库里提供了做服务器编程能用到的99%的东西。PHP 的 PECL 扩展库里你想要的任何的功能。

- PHP 有超过20年的历史，生态圈是非常大的，在 Github 可以找到很多代码。

PHP 的缺点如下。

- 性能比较差，因为毕竟是动态脚本，不适合做密集运算，如果同样用 PHP 写再用 C++ 写，PHP 版本要比它差一百倍。

- 函数命名规范差，这一点大家都是了解的，PHP 更讲究实用性，没有一些规范。一些函数的命名是很混乱的，所以每次你必须去翻 PHP 的手册。

- 提供的数据结构和函数的接口粒度比较粗。PHP 只有一个 Array 数据结构，底层基于 HashTable。PHP 的 Array 集合了 Map、Set、Vector、Queue、Stack、Heap 等数据结构的功能。另外 PHP 有一个 SPL 提供了其他数据结构的类封装。

综上，可以说 PHP：

- 更适合偏实际应用层面的程序，业务开发、快速实现的利器。

- 不适合开发底层软件。

- 使用 C/C++、Java、Golang 等静态编译语言作为 PHP 的补充，动静结合。

- 借助 IDE 工具实现自动补全、语法提示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4f74cd0e6.png" alt="图3  PHP的优点与缺点" title="图3  PHP的优点与缺点" />

图3  PHP 的优点与缺点

### PHP 的 Swoole 扩展

基于上面的扩展使用纯 PHP 就可以完全实现异步网络服务器和客户端程序。但是想实现一个类似于多 I/O 线程，还是有很多繁琐的编程工作要做，包括如何来管理连接，如何来保证数据的收发原则性，网络协议的处理。另外 PHP 代码在协议处理部分性能是比较差的，所以我启动了一个新的开源项目 Swoole，使用 C 语言和 PHP 结合来完成了这项工作。灵活多变的业务模块使用 PHP 开发效率高，基础的底层和协议处理部分用 C 语言实现，保证了高性能。它以扩展的方式加载到了 PHP 中，提供了一个完整的网络通信的框架，然后 PHP 的代码去写一些业务。它的模型是基于多线程 Reactor+ 多进程 Worker，既支持全异步，也支持半异步半同步。

Swoole 的一些特点：

- Accept 线程，解决 Accept 性能瓶颈和惊群问题；

- 多 I/O 线程，可以更好地利用多核；

- 提供了全异步和半同步半异步两种模式；

- 处理高并发 I/O 的部分用异步模式；

- 复杂的业务逻辑部分用同步模式；

- 底层支持了遍历所有连接、互发数据、自动合并拆分数据包、数据发送原子性。

Swoole 的进程/线程模型如图4所示，图5为 Swoole 程序的执行流程。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4faae5648.png" alt="图4  Swoole的进程/线程模型" title="图4  Swoole的进程/线程模型" />

图4  Swoole 的进程/线程模型

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d4fdbb0c8e.png" alt="图5  Swoole程序的执行流程" title="图5  Swoole程序的执行流程" />

图5  Swoole 程序的执行流程

#### 使用 PHP+Swoole 扩展实现异步通信编程

实例代码在 https://github.com/swoole/swoole-src 主页查看。

TCP 服务器与客户端

异步 TCP 服务器：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d500d75d7d.png" alt="代码4" title="代码4" />

代码4

在这里 new swoole_server 对象，然后参数传入监听的 HOST 和 PORT，然后设置了3个回调函数，分别是 onConnect 有新的连接进入、onReceive 收到了某一个客户端的数据、onClose 某个客户端关闭了连接。最后调用 start 启动服务器程序。Swoole 底层会根据当前机器有多少 CPU 核数，启动对应数量的 Reactor 线程和 Worker 进程。

异步客户端：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d503eae2ae.png" alt="代码5" title="代码5" />

代码5

客户端的使用方法和服务器类似只是回调事件的有4个，onConnect 成功连接到服务器，这时可以去发送数据到服务器。onError 连接服务器失败。onReceive 服务器向客户端连接发送了数据。onClose 连接关闭。

设置完事件回调后，发起 Connect 到服务器，参数是服务器的 IP，PORT 和超时时间。

同步客户端：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d50695f9c2.png" alt="代码6" title="代码6" />

代码6

同步客户端不需要设置任何事件回调，它没有 Reactor 监听，是阻塞串行的。等待 I/O 完成才会进入下一步。

异步任务：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d5096b8239.png" alt="代码7" title="代码7" />

代码7

异步任务功能用于在一个纯异步的 Server 程序中去执行一个耗时的或者阻塞的函数。底层实现使用进程池，任务完成后会触发 onFinish，程序中可以得到任务处理的结果。比如一个 IM 需要广播，如果直接在异步代码中广播可能会影响其他事件的处理。另外文件读写也可以使用异步任务实现，因为文件句柄没办法像 Socket 一样使用 Reactor 监听。因为文件句柄总是可读的，直接读取文件可能会使服务器程序阻塞，使用异步任务是非常好的选择。

异步毫秒定时器：

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d50cdf3bbb.png" alt="代码8" title="代码8" />

代码8

这两个接口实现了类似 JavaScript 的 setInterval、setTimeout 函数功能，可以设置在 n 毫秒间隔实现一个函数或n毫秒后执行一个函数。

异步 MySQL 客户端见代码9。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d50fd32d25.png" alt="代码9" title="代码9" />

代码9

Swoole 还提供一个内置连接池的 MySQL 异步客户端，可以设定最大使用 MySQL 连接数。并发 SQL 请求可以复用这些连接，而不是重复创建，这样可以保护 MySQL 避免连接资源被耗尽。

异步 Redis 客户端见代码10。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d513f7914b.png" alt="代码10" title="代码10" />

代码10

异步的 Web 程序，如代码11所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d5172de181.png" alt="代码11" title="代码11" />

代码11

程序的逻辑是从 Redis 中读取一个数据，然后显示 HTML 页面。使用 AB 压测性能如图6所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d51a7419f1.png" alt="图6  AB压测性能结果" title="图6  AB压测性能结果" />

图6  AB 压测性能结果

同样的逻辑在 PHP-FPM 下的性能测试结果如图7。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d51e064c12.png" alt="图7  PHP-FPM下的性能测试结果" title="图7  PHP-FPM下的性能测试结果" />

图7  PHP-FPM 下的性能测试结果

WebSocket 程序见代码12。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d52147670f.png" alt="代码12" title="代码12" />

代码12

Swoole 内置了 WebSocket 服务器，可以基于此实现 Web 页面主动推送的功能，比如 WebIM。有一个开源项目可以作为参考：https://github.com/matyhtf/php-webim

PHP+Swoole 协程

异步编程一般使用回调方式，如果遇到非常复杂的逻辑，可能会层层嵌套回调函数。协程就可以解决此问题，可以顺序编写代码，但运行时是异步非阻塞的。腾讯的工程师基于 Swoole 扩展和 PHP5.5 的 Yield/Generator 语法实现类似于 Golang 的协程，项目名称为 TSF（Tencent Server Framework），开源项目地址：https://github.com/tencent-php/tsf。目前在腾讯公司的企业 QQ、QQ 公众号项目以及车轮忽略的查违章项目有大规模应用 。

TSF 使用也非常简单，下面调用了3个 I/O 操作，完全是串行的写法。但实际上是异步非阻塞执行的。TSF 底层调度器接管了程序的执行，在对应的 I/O 完成后才会向下继续执行。

<img src="http://ipad-cms.csdn.net/cms/attachment/201606/574d5248c697c.png" alt="代码13" title="代码13" />

代码13

#### 在树莓派上使用 PHP+Swoole

PHP 和 Swoole 都可以在 ARM 平台上编译运行，所以在树莓派系统上也可以使用 PHP+Swoole 来开发网络通信的程序。
