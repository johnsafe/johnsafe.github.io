## iOS 开发下的函数响应式编程——美团函数响应式开发实践

文/ 臧成威

>随着移动互联网的蓬勃发展，iOS App 的复杂度呈指数增长。美团·大众点评两个 App 随着用户量不断增加、研发工程师数量不断增多，可用性的要求也随之不断提升。在这样一个背景下，我们面临了很多的问题和挑战。iOS 工程师们想出了很多的策略和方针来应对，引入函数响应式编程就是其中重要的一环。

### 函数响应式编程简介

函数式编程想必你一定听过，但响应式编程的说法就不大常见了。与响应式编程对应的命令式编程就是大家所熟知的一种编程范式，我们先来看一段代码：

```
int a = 3;
    int b = 4;
    int c = a + b;
    NSLog(@"c is %d", c); // 7
    a = 5;
    b = 7;
    NSLog(@"c is %d", c); // 仍然是7
```

命令式编程就是通过表达式或语句来改变状态量，例如 c = a + b 就是一个表达式，它创建了一个名称为 c 的状态量，其值为 a 与 b 的和。下面的 a = 5是另一个语句，它改变了 a 的值，但这时 c 是没有变化的。所以命令式编程中 c = a + b 只是一个瞬时的过程，而不是一个关系描述。在传统的开发中，想让 c 跟随 a 和 b 的变化而变化是比较困难的。而让c的值实时等于 a 与 b 的和的编程方式就是“响应式编程”。

实际上，在日常中我们会经常使用响应式编程。最典型的例子就是 Excel，当我们在一个 B1 单元格上书写一个公式“=A1+5”时，便声明了一种对应关系，每当 A1 单元格发生变化时，单元格 B2 都会随之改变。

iOS 开发中也有响应式编程的典型例子，例如 Autolayout。我们通过设置约束描述了各个视图的位置关系，一旦其中一个变化，另一个就会响应其变化。当然，类似的例子还有很多。

函数响应式编程（Functional Reactive Programming，以下简称 FRP，）正是在函数式编程的基础之上，增加了响应式的支持。简单来讲，FRP 是基于异步事件流进行编程的一种编程范式。针对离散事件序列进行有效的封装，利用函数式编程的思想，满足响应式编程的需要。

### iOS 项目的函数响应式编程选型

很长一段时间以来，iOS 项目并没有很好的FRP支持，直到 iOS 4.0 SDK 中增加了 Block 语法才为函数式编程提供了前置条件，FRP 开源库也逐步健全起来。最先与大家见面的莫过于 ReactiveCocoa 这个库了，ReactiveCocoa 是 GitHub 在制作客户端时开源的一个副产物，缩写为 RAC。它是 Objective-C 语言下 FRP 思想的一个优秀实例，后续版本也支持了 Swift 语言。

除了 RAC，RxSwift 是 ios 下 FRP 开发的另一个选择。表1是各开源库的对比。由于历史原因美团 App 使用 RAC2.5版本，下文也基于2.5来介绍。

表1 iOS 下几种 FRP 库的对比

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281810af34a.png" alt="表1 iOS下几种FRP库的对比" title="表1 iOS下几种FRP库的对比" />

你可能会问，为什么需要在 iOS 项目中引入 RAC 这样厚重的库呢？iOS 的项目主要以客户端目为主，主要的业务场景就是进行页面交互和与服务器拉取数据，这里会包含多种事件和异步处理逻辑。FRP 本身就是面向事件流的一种开发方式，又擅长处理异步逻辑。所以从逻辑上是满足 iOS 客户端业务需要的。

### 一步一步进行函数响应式编程

众所周知，FRP 的学习存在一定门槛，想必这也是大家对 FRP、ReactiveCocoa 这些概念比较畏惧的主要原因。美团 App 在推行 FRP 的过程中，是采用分步骤的形式，逐步演化，可以分为初探、入门、提高、进阶这样四个阶段。

#### 初探

2014年5月我们第一次将 ReactiveCocoa 纳入到工程中，当时 iOS 工程师数量还不是很多，但是已经遇到了写法不统一、代码量膨胀等问题了。写法不统一聚焦在回调形式上，iOS 中的回调方式有非常多的种类：UIKit 主要进行的事件处理 target-action、跨类依赖推荐的 delegate 模式、iOS 4.0纳入的 block、利用通知中心（Notifcation Center）进行松耦合的回调、利用键值观察（Key-Value Observe，简称 KVO）进行的监听。由于场景不同，选用的规则也不尽相同，并且我们没有办法很好地界定什么场景该写什么样的回调。

ReactiveCocoa 恰好能以统一的形式来解决跨类调用的问题，也包装了 UIKit 事件处理、Notifcation Center、KVO 等常见的场景，使其代码风格高度统一。使用 ReactiveCocoa 后，上述的几种回调都可以写成如下形式：

```
// 代替target-action
    [[self.confirmButton rac_signalForControlEvents:UIControlEventTouchUpInside]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
     
// 代替delegate
    [[self.scrollView rac_signalForSelector:@selector(scrollViewDidScroll:) fromProtocol:@protocol(UIScrollViewDelegate)]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
     
// 代替block
    [[self asyncProcess]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     } error:^(NSError *error) {
        // 错误处理写到这里
     }];
     
// 代替notification center
    [[[NSNotificationCenter defaultCenter] rac_valuesForKeyPath:@"Some-key" observer:nil]
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
 
// 代替KVO
    [RACObserve(self, userReportString)
     subscribeNext:^(id x) {
        // 回调内容写在这里
     }];
```

通过观察代码不难发现，ReactiveCocoa 使得不同场景下的代码样式高度统一，使我们在书代码、维护码、阅读代码方面的效率大大提高。

经过研究，我们也发现使用 RAC(target, key)宏可以更好地组织代码形式，利用 filter:和 map:来代替原有的代码，达到更好复用。虽然代码行数有一定的增加，但结构更加清晰，复用性也更好。

```
// 旧写法
    @weakify(self)
    [[self.textField rac_newTextChannel]
     subscribeNext:^(NSString *x) {
         @strongify(self)
         if (x.length > 15) {
             self.confirmButton.enabled = NO;
             [self showHud:@"Too long"];
         } else {
             self.confirmButton.enabled = YES;
         }
         self.someLabel.text = x;
     }];
// 新写法
    RACSignal *textValue = [self.textField rac_newTextChannel];
    RACSignal *isTooLong = [textValue                          map:^id(NSString *value) {
                                return @(value.length > 15);
                            }];
    RACSignal *whenItsTooLongMessage = [[isTooLong
                                         filter:^BOOL(NSNumber *value) {                                        return @(value.boolValue);                                         }]                                       mapReplace:@"Too long"];
    [self rac_liftSelector:@selector(showHud:) withSignals:whenItsTooLongMessage, nil];
    RAC(self.confirmButton, enabled) = [isTooLong not];
    RAC(self.someLabel, text) = textValue;
```

综上所述，在这一阶段，我们主要以回调形式的统一为主，不断尝试合适的代码形式来表达绑定这种关系，也寻找一些便捷的小技巧来优化代码。

#### 入门

单纯解决回调风格的统一和树立绑定的思维是远远不够的，代码中更大的问题在于共享变量、异步协同以及异常传递的处理。 

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/5728182e9e715.png" alt="图1 美团App首页" title="图1 美团App首页" />

图1 美团 App 首页

就拿美团 App 的首页来讲（如图1所示），我们可以看到上面包含很多的区块，而各个区块的访问接口不尽相同，但渲染的逻辑却又多种多样：

- 有的需要几个接口都返回后才能统一渲染；

- 有的需要一个接口返回后，根据返回的内容决定后续的接口访问，最终才能渲染；

- 有的则是几个接口按照返回顺序依次渲染。 

这就导致了在处理这些逻辑的时候，需要很多的异步手段，利用中间变量来储存状态，每次返回时又判断这些状态决定渲染的逻辑。更糟糕的是，有时对于同时请求多个网络接口，某些出现了网络错误，异常处理也变得越来越复杂。

随着对 ReactiveCocoa 理解的加深，我们意识到使用信号的组合等“高级”操作可以帮助我们解决很多的问题。例如 merge:操作可以解决依次渲染的问题，zip:操作可以解决多个接口都返回后再渲染的问题，flattenMap:可以解决接口串联的问题。示例代码如：

```
// 依次渲染
    RACSignal *mergedSignal = [RACSignal merge:@[[self fetchData1],                                                [self fetchData2]]];    
// 接口都返回后一起渲染
    RACSignal *zippedSignal = [RACSignal zip:@[[self fetchData1],
                                               [self fetchData3]]];    
// 接口串联
    @weakify(self)
    RACSignal *finalSignal = [[self fetchData4]                             flattenMap:^RACSignal *(NSString *data4Result) {
                                  @strongify(self)
                                  return [self fetchData5:data4Result];
                              }];
```

这段代码没有用到一个中间状态变量，我们通过这几个“魔法接口”神奇地将逻辑描述了出来。FRP 具备这样一个特点，信号因为进行组合从而得到了一个数据链，而数据链的任一节点发出错误信号，都可以顺着这个链条最终交付给订阅者。这就正好解决了异常处理的问题（如图2所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/5728184c179fa.png" alt="图2 错误传递链" title="图2 错误传递链" />

图2 错误传递链

由于此项特性，我们可以不再关注错误在哪个环节，只需等待订阅的时候统一处理即可。我们也找到了很多的方法用来更好地支持异常的处理。例如try:、catch:、catchTo:、tryMap:等。简单举例：

```
// 尝试判断并捕捉异常
    RACSignal *signalForSubscriber =
     
      [[[[[self fetchData1]
          try:^BOOL(NSString *data1,
                    NSError *__autoreleasing *errorPtr) {
              if (data1.length == 0) {
                  *errorPtr = [NSError new];
                  return YES;
              }
              return NO;
          }]
         flattenMap:^RACStream *(id value) {
             @strongify(self)
             return [self fetchData5:value];
         }]
        try:^BOOL(id value,
                  NSError *__autoreleasing *errorPtr) {
            if (![value isKindOfClass:[NSString class]]) {
                *errorPtr = [NSError new];
                return YES;
            }
            return NO;
        }]
       catch:^RACSignal *(NSError *error) {
           return [RACSignal return:error.domain];
       }];
```

在初探和入门这两个阶段，还只是谨慎地进行小尝试，以代码简化为目的，使用 ReactiveCocoa 这个开源框架的一些便利功能来优化代码。在代码覆盖程度上尽量只在模块内部使用，避免跨层。

#### 提高

随着对 ReactiveCocoa 的理解不断加深，我们并不满足于简单的尝试，而是开始在更多的场景下使用，并体现一定的 FRP 思想。这个阶段最具代表性的实践就是与 MVVM 架构的融合了，体现了 FRP 响应式的思想。

在 MVVM 架构中，最为关键的一环莫过于 ViewModel 层与 View 层的绑定了，我们的主角 FRP 恰好可以解决绑定问题，同时还能处理跨层错误处理的问题。

自初探阶段开始，我们就开始使用 RAC(target, key)这样的一个宏来表述绑定的关系，并且使用一些简单的信号转换使得原始信号满足视图渲染的需求。在引入 MVVM 架构后，我们将之前的经验利用起来，并且使用了 RACChannel、RACCommand 等组件来支持 MVVM。

```
// 单向绑定
    RAC(self.someLabel, text) = RACObserve(self.viewModel, someProperty);
    RAC(self.scrollView, hidden) = self.viewModel.someSignal;
    RAC(self.confirmButton, frame) = [self.viewModel.someChannel                                      map:^id(NSString *str) {                                          CGRect rect = CGRectMake(0, 0, 0, str.length * 3);                                         return [NSValue valueWithCGRect:rect];                                      }];  
// 双向绑定
    RACChannelTo(self.someLabel, text) = RACChannelTo(self.viewModel, someProperty);
    [self.textField.rac_newTextChannel subscribe:self.viewModel.someChannel];
    [self.viewModel.someChannel subscribe:self.textField.rac_newTextChannel];
    RACChannelTo(self, reviewID) = self.viewModel.someChannel;    
// 命令绑定
    self.confirmButton.rac_command = self.viewModel.someCommand;   
    RAC(self.textField, hidden) = self.viewModel.someCommand.executing;
    [self.viewModel.someCommand.errors
     subscribeNext:^(NSError *error) {
         // 错误处理在这里
     }];
```

绑定只是提供了上层的业务逻辑，更为重要的是，FRP 的响应式范式恰如其分地体现在 MVVM 中。一个 MVVM 中 View 就会响应 ViewModel 的变化。我们利用图3来简单地分析一下。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/5728187d2e2a3.png" alt="图3 MVVM示意图" title="图3 MVVM示意图" />

图3 MVVM 示意图

图3列出了 View-ViewModel-Model 的大致关系，View 和 ViewModel 间通过 RACSignal 来进行单向绑定，通过 RACChannel 来双向绑定，通过 RACCommand 进行执行过程的绑定。ViewModel 和 Model 间通过 RACObserve 进行监听，通过 RACSignal 进行回调处理，可能也直接调用方法。Model 有自身的数据业务逻辑，包含请求 Web Service 和进行本地持久化。

响应式的体现就在于 View 从一开始就是“声明”了与 ViewModel 间的关系，就如同 A3 单元格声明其“=A2+A1”一样。一旦后续数据发生变化，就按照之前的约定响应，彼此之间存在一层明确的定义。View 在业务层面也得到了极大简化。具体的数据流动就如图4形式。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/572818b8c91f7.png" alt="图4 MVVM的数据流向示意" title="图4 MVVM的数据流向示意" />

图4 MVVM 的数据流向示意

从图4可以看出用户点击 Button 事件，view 层并不需要做特殊处理。只是触发了 ViewModel 之前绑定的 command，再由 command 做相应处理。这是与 MVC 中 ViewController 和 Model 的关系截然不同。FRP 的响应式范式很好地帮助我们实现了此类需求。之前虽然也提到过错误处理，但只是小 规模地在模块内使用，对外并不会以 RACSignal 的形式暴露。而这个阶段，我们也尝试了层级间通过  RACSignal 来进行信息的传递。这也自然可以应用上 FRP 异常处理的优势。图5体现了一个按钮绑定 RACCommand 收到错误后的数据流向。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/572818cf5c529.png" alt="图5 MVVM中异常处理" title="图5 MVVM中异常处理" />

图5 MVVM 中异常处理

除了 MVVM 框架的应用，这个阶段也利用 FRP 解决另外的一个大问题。那就是多线程。如果你做过异步拉取数据主线程渲染，那么一定很熟悉子线程的回调结果转移到主线程的过程。这种操作不但繁琐、重复，还容易出错。

RAC 提供了很多便于处理多线程的组件，核心类为 RACScheduler，使得我们可以方便地通过 subscirbeOn:、deliverOn:方法进行线程控制。

```
// 多线程控制
    RACScheduler *backgroundScheduler = [RACScheduler scheduler];
RACSignal *testSignal = [[RACSignal                             createSignal:^RACDisposable *(id<racsubscriber> subscriber) {
 // 本例中，这段代码会确保运行在子线程                                [subscriber sendNext:@1];                                [subscriber sendCompleted];
  return nil;
                             }]
subscribeOn:backgroundScheduler]; 
    [[RACScheduler mainThreadScheduler]
     schedule:^{
// 这段代码会在下次Runloop时在主线程中运行
         [[testSignal
          deliverOn:[RACScheduler mainThreadScheduler]]
            subscribeNext:^(id x) {
// 虽然信号的发出者是在子线程中发出消息的
// 但是接收者确可以确保在主线程接收消息              
// 主线程可以做你想做的渲染工作了！
   }
 ];
}];</racsubscriber>
```

这也算是大跃进的一个阶段，随着 MVVM 的框架融入，跨层的使用 RAC 使得代码整体使用FRP的比重大幅提高，全员对 FRP 的熟悉程度和思想的理解也变得深刻了许多。同时也真正使用了响应式的一些思想和特性来解决实际问题而不再是纸上空谈。我们也在这个阶段挖掘了更多的 RAC 高级组件，继续代码优化之路。

#### 进阶

在大规模使用 FRP 后，自然积蓄了很多问题。这一阶段面临的问题是 RAC 的大规模应用，使得代码中包含了大量的框架性质的代码。

```
冗余的代码
    [[self fetchData1]
     try:^BOOL(id value,
               NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSString class]]) {
             return YES;
         }
         *errorPtr = [NSError new];
         return NO;
     }];    
    [[self fetchData2]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someKey"]) {
                 return value[@"someKey"];
             }
             // 并没有一个名为“someKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];   
    [[self fetchData3]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someOtherKey"]) {
                 return value[@"someOtherKey"];
             }
             // 并没有一个名为“someOtherKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
```

我们可以看到功能非常近似，内容稍有不同的部分重复出现，很多同学在实际的开发中也并没有太好地优化它们。这时候函数式编程就可以派上用场了。函数式编程是一种良好的编程范式，我们在这里主要利用它的几个特点：高阶函数、不变量和迭代。

先来看高阶函数，高阶函数是入参是函数或者返回值是函数的函数。说起来虽然有些拗口，但实际上在 iOS 开发中司空见惯，例如典型的订阅其实就是一个高阶函数的体现。

我们更关心的是返回值是函数的函数，这是上面冗长的代码解决之道。代码中会发现一些相同的逻辑，例如类型判断。我们就可以先做一个这样的小函数：

```
typedef BOOL (^VerifyFunction)(id value);
 
VerifyFunction isKindOf(Class aClass)
{
    return ^BOOL(id value) {
        return [value isKindOfClass:aClass];
    };
}
```

很简单对不对！只要把一个类型传进去，就会得到一个用来判断某个对象是否是这个类型的函数。细心的读者会发现我们实际要的是一个入参为对象和一个 NSError 对象指针的指针类型，返回值是布尔类型的 block，但这个只能返回入参是对象的，显然不满足条件。很多人第一个想到的就是把这个函数改成返回参数为两个参数返回值为布尔类型的 block，但函数式的解决方法是新增一个函数。

```
typedef BOOL (^VerifyAndOutputErrorFunction)(id value, NSError **error);
 
VerifyAndOutputErrorFunction verifyAndOutputError(VerifyFunction verify,
                                                  NSError *outputError)
{
    return ^BOOL(id value, NSError **error) {
        if (verify(value)) {
            return YES;
        }
        *error = outputError;
        return NO;
    };
}
```

一个典型的高阶函数，入参带有一个 block，返回值也是一个 block，组合起来就可以把刚才的几个 try:代码段优化。可能你会问，为什么要搞成两个呢，一个不是更好？搞成两个的好处就在于，我们可以将任意的 VerifyFunction 类型的 block 与一个 outputError 相结合，来返回一个我们想要的 VerifyAndOutputErrorFunction 类型 block，例如增加一个判断 NSDictionary 是否包含某个 Key 的 VerifyFunction。下面给出一个优化后的代码，大家可以仔细思考下：

```
// 可以高度复用的函数
typedef BOOL (^VerifyFunction)(id value);
VerifyFunction isKindOf(Class aClass)
{
    return ^BOOL(id value) {
        return [value isKindOfClass:aClass];
    };
}
VerifyFunction hasKey(NSString *key)
{
    return ^BOOL(NSDictionary *value) {
        return value[key] != nil;
    };
}
typedef BOOL (^VerifyAndOutputErrorFunction)(id value, NSError **error);
VerifyAndOutputErrorFunction verifyAndOutputError(VerifyFunction verify,                                                NSError *outputError)
{
    return ^BOOL(id value, NSError **error) {
        if (verify(value)) {
            return YES;
        }
        *error = outputError;
        return NO;
    };
}
typedef id (^MapFunction)(id value);
 
MapFunction dictionaryValueByKey(NSString *key)
{
    return ^id(NSDictionary *value) {
        return value[key];
    };
}
// 与本例关联比较大的函数
typedef id (^MapAndOutputErrorFunction)(id value, NSError **error);
MapAndOutputErrorFunction transferToKeyChild(NSString *key)
{
    return ^id(id value, NSError **error) {
        if (hasKey(key)(value)) {
            return dictionaryValueByKey(key)(value);
        } else {
            *error = [NSError new];
            return nil;
        }
    };
};
- (void)oldStyle
{
    // 冗余的代码
    [[self fetchData1]
     try:^BOOL(id value,
               NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSString class]]) {
             return YES;
         }
         *errorPtr = [NSError new];
         return NO;
     }];
     
    [[self fetchData2]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someKey"]) {
                 return value[@"someKey"];
             }
 // 并没有一个名为“someKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
         // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];
     
    [[self fetchData3]
     tryMap:^id(NSDictionary *value, NSError *__autoreleasing *errorPtr) {
         if ([value isKindOfClass:[NSDictionary class]]) {
             if (value[@"someOtherKey"]) {
return value[@"someOtherKey"];
             }
  // 并没有一个名为“someOtherKey”的key
             *errorPtr = [NSError new];
             return nil;
         }
 // 这不是一个Dictionary
         *errorPtr = [NSError new];
         return nil;
     }];
- (void)newStyle
{
    VerifyAndOutputErrorFunction isDictionary = 
      verifyAndOutputError(isKindOf([NSDictionary class]),
                        NSError.new);
    VerifyAndOutputErrorFunction isString =      
      verifyAndOutputError(isKindOf([NSString class]),
                           NSError.new);
  
    [[self fetchData1]
     try:isString];
    [[[self fetchData2]
      try:isDictionary]
      tryMap:transferToKeyChild(@"someKey")];
     
    [[[self fetchData3]
      try:isDictionary]
     tryMap:transferToKeyChild(@"someOtherKey")];
}
```

虽然代码有些多，但从 newStyle 函数的结果来看，实际的业务代码非常简洁，而且还抽离出很多可复用的小函数。我们甚至通过这种范式在某些业务场景简化了超过50%的代码量。

除此之外，我们还尝试用迭代来进一步减少临时变量。为什么要减少临时变量呢？因为我们想要遵循不变量原则，这是函数式编程的一个特点。试想下如果我们都是使用一些不变量，就不再会有那么多异步锁和痛苦的多线程问题了。基于以上考虑，我们要求工程师尽量在开发的过程中减少使用变量，从而锻炼用更加函数式的方式来解决问题。例如下面这个简单问题，实现一个每秒发送值为0 1 2 3 … 100的递增整数信号，实现的方法可以是这样：

```
- (void)countSignalOldStyle
{
    RACSignal *signal =
    [RACSignal createSignal:^RACDisposable *(id<racsubscriber> subscriber) {
        RACDisposable *disposable = [RACDisposable new];
        __block int i = 0;
        __block void (^recursion)();
        recursion = ^{
            if (disposable.disposed || i > 100) {
                return;
            }
            [subscriber sendNext:@(i)];
            ++i;
            [[RACScheduler mainThreadScheduler]
             afterDelay:1 schedule:recursion];
        };
        recursion();
        return disposable;
    }];
}</racsubscriber>
```

这样的代码不但用了 block 自递归，还用了一个闭包的 i 变量。i 变量也在数次递归中进行了修改。代码不易理解且 block 自递归会存在循环引用。使用迭代和不变量的形式如下所示。

```
- (void)countSignalNewStyle
{
    RACSignal *signal =
    [[[[[[RACSignal return:@1]
         repeat] take: 100]
       aggregateWithStart:@0 reduce:^id(NSNumber *running,
                                        NSNumber *next) {
           return @(running.integerValue + next.integerValue);
       }]
      map:^id(NSNumber *value) {
          return [[RACSignal return:value]
                  delay:1];
      }]
     concat];
}
```

解法是这样的，先用固定返回1的信号生成无限重复信号，取前100个值，然后用迭代方法，产生一个递增的迭代，再将发送的密集递增信号转成一个延时1秒的子信号，最后将子信号进行连接。

### 结束语

FRP 不仅可以解决项目中实际遇到的很多问题，也能锻炼更好的工程师素养，希望大家能够掌握起来，用 FRP 的思想来解决更多实际问题。社区和开源库也需要大家的不断投入。