## 现代 C++ 实现万能函数容器

文/祁宇

>如何实现一个通用容器可以保存所有类型的函数并实现调用？本文通过 C++ 11/14的一些新特性和模版元技巧解决这一技术难题。

把一个函数放到普通的容器中是容易的，比如这样：

```
#include <map>
#include <string>

void foo(int){}
int main()
{
std::map<std::string,
std::function<void(int)> map;
map.emplace("test", &foo);
map["test"](1);
}
```

但是这个map中可以存放的函数类型已确定，无法存放其他类型的函数，比如 int foo(std::string)就无法保存到 map 中，因为函数的类型和 map 定义的函数类型不一致。那么，有什么办法能将所有函数都保存到某个容器，然后在需要时实调用呢？看起来是个不可能完成的任务，但对于现代 C++ 来说却不是一个很困难的问题。解决的关键是如何把函数的类型擦除，然后在获取时又把函数还原出来调用。关于类型擦除，C++ 有不少方法。

### 类型擦除

下文简单归纳 C++ 中的类型擦除方式。

- 通过多态来擦除类型

这种方式是通过将派生类型隐式转换成基类型，再通过基类去多态调用行为，在这种情况下，不用关心派生类的具体类型，只需要以一种统一方式去做不同的事情。不仅仅可以多态调用，还使我们的程序具有良好的可扩展性。然而这种方式仅仅是部分的类型擦除，因为基类型仍然存在，并且这种类型擦除方式还必须是继承方式才可以，然而继承使得两个对象强烈耦合在一起，正是因为这些缺点，通过多态来擦除类型的方式有较多局限性，效果也不好。

- 通过模版来擦除类型

这种方法本质上是把不同类型的共同行为进行了抽象，这时不同类型彼此之间不需要通过继承这种强耦合的方式去获得共同的行为了，仅仅通过模板就能获取共同行为，降低了不同类型之间的耦合，是一种很好的类型擦除方式。然而，这种方式虽然降低了对象间的耦合，但具体类型始终需要指定，并没有消除类型，导致无法保存到容器当中。

- 通过类型容器来擦除类型

boost.variant 是一个类型容器，variant 可以包含多种类型，从而让用户获得了一种统一的类型，并且不同类型的对象之间并没有耦合关系，实现了类型擦除的目标。但是也有一些局限性，variant 的类型必须事先定义好，只能容纳那些事先定义的类型，如果事先无法知道这些具体的类型，是无法动态定义新 variant 类型的。

- 通过通用类型擦除

C++ 不像 C#、Java 等语言一样，所有的对象都继承自 object 类型，object 是所有类型的基类，然而 C++ 中没有这种 object 基类。不过 boost.any 可以用来实现彻底的类型擦除。其基本用法如下：

```
unordered_map<string, boost::any> m_
creatorMap;
m_creatorMap.insert(make_pair(strKey, new
T)); //T may be any type
boost::any obj = m_creatorMap[strKey];
T t = boost::any_cast<T>(obj);
```

需要注意的是，虽然 boost.any 能彻底擦除类型，但是在取值，即将对象从 any 还原出来的时候需要具体的类型，这导致 boost.any 仍然存在一些局限。

通过这几种典型的类型擦除方式中的某一种方式能实现“将所有类型的函数都放入一个容器”这一目标吗？通过多态或者模版的方法都是行不通的，因为函数类型不能继承于某一个基类，而容器又不能保存一个带模版参数的泛型函数。通过 variant 也是不可行的，因为 variant 的具体类型需要事先定义，然而函数类型有无数种，不可能事先定义。那么 boost.any 可以吗？把函数先转换为 any，保存到容器当中，然后在取值时将 any 对象转换为函数。这样看来似乎是可以的，但存在几个问题：

- 函数的一些基本信息丢失，将函数转换为 any 的时候，类型信息丢失了，可能是普通函数，也可能是成员函数，还可能是 lambda 表达式等可调用对象，但是这个类型信息完全隐藏在 any 当中了。另外，函数的形参信息丢失了，形参可能是引用，也可能是常量引用，这些信息无法再从 any 中获取了。

- 函数从 any 还原为函数对象时无法获取实际的函数类型信息，也许有人认为可以通过将可调用对象统一转换为 std::function 的方式来避免函数类型丢失的问题，这虽然是一种可行的办法，但是形参类型却无法从 boost.any 中获取，如果传入的实参和实际的形参类型不一致，会导致 bad_cast 异常。除非在取值的时候带上实际的形参类型，然而这又导致接口使用不便，如果接口参数较多会显得非常冗长，接口的易用性和代码的可读性大幅降低。

在分析了几种典型的类型擦除技术之后，发现它们都不满足“将所有类型的函数都放入一个容器”的目标。那么这个目标应该如何实现呢？通过现代 C++ 的一些新特性和模版元编程技巧我们完全可以实现这一目标。

### 将所有类型函数都放入一个容器

解决这个问题，需要分两步，第一步还是类型擦除，第一步实现之后还要实现类型还原。

#### 特别的类型擦除

具体思路是将函数类型隐藏起来放到一个 std::function 中，这个 std::function 是一个具体的类型，可以保存到容器中。如何将函数类型隐藏起来放入 function 呢？通过 bind 和一个模版类，下面是具体的实现代码：

```
template<typename Function>
struct invoker
{
static inline void apply(const
Function& func, void* bl)
{

}
};
template<typename Function>
void register_handler(std::string const
& name, const Function& f)
{
using std::placeholders::_1;
using std::placeholders::_2;
std::map<std::string,
std::function<void(void*)>> invokers;
this->invokers[name] = { std::bind(&
invoker<Function>::apply, f, _1) };
}
```

首先定义了一个 map，值类型为 std::function，这个明确的 function 类型是可以放入到容器当中的，接着借助 std::bind 将模版类 invoker 的静态成员函数 apply 绑定起来，bind 的作用有两个：第一，将 apply 成员函数绑定为 std::function 类型，从而实现类型擦除以保存到容器中；第二，bind 的同时将函数类型作为 invoker 的模版参数保存起来，保存函数类型的原因是：保证类型擦除时，函数的类型不会丢，也为了将函数还原出来的时候提供必需的函数返回类型和形参类型。

在解决了类型擦除的问题之后，接着就要实现第二步了：类型还原。

#### 类型还原

类型还原的思路是将参数传给 std::function，在 function 绑定的 invoker::apply 中解析参数和获取函数的类型信息，还原函数并实现函数调用。下面是具体的实现代码：

```
template <typename ... Args>Technology 技术
111
void call(const std::string& name,
Args&& ... args)
{
auto it = invokers_.find(name);
if (it == invokers_.end())
return{};
auto args_tuple = std::make_tuple(st
d::forward<Args>(args)...);

char data[sizeof(std::tuple<Ar
gs...>)];
std::tuple<Args...>* tp = new (data)
std::tuple<Args...>;
*tp = args_tuple;

it->second(tp);
}
template<typename Function>
struct invoker
{
static inline void apply(const
Function& func, void* bl)
{
using tuple_type = typename
function_traits<Function>::tuple_type;
const tuple_type* tp = static_
cast<tuple_type*>(bl);
call(func, *tp);
}
};
```

首先从容器中查找对应的 std::function，接着将实参放入 std::tuple 中，并通过 placement new 将 tuple 转换为 void* 指针，然后将 function 和 tuple 转换而来的 void* 指针传入到函数 apply 中，apply 函数中先根据第一步保存在 invoker 中的函数类型，借助 function\_traits 萃取出参数的 tuple 类型（function\_traits 的具体实现可以参考我的 github 上的代码）。最后根据 function 和 tuple 实现函数调用，下面是 call 的具体实现：

```
template<typename F, typename ...
Args>
void call(const F& f, const
std::tuple<Args...>& tp)
{
call_helper(f, std::make_index_
sequence<sizeof... (Args)>{}, tp);
}
template<typename F, size_t... I,
typename ... Args>
static decltype(auto) call_
helper(const F& f, const
std::index_sequence<I...>&, const
std::tuple<Args...>& tup)
{
return f(std::get<I>(tup)...);
}
```

call 主要是借助 C++ 11的新特性可变模版参数和 C++ 14的 std::index\_sequence 来实现将 tuple 编程函数的实参，从而实现函数调用。

### 一个应用实例

前面实现了能将所有函数放入容器并实现调用的简单示例，可以用这个技术实现一个本地的消息总线，消息总线一般用于解耦对象之间的调用关系，将调用者和接受者解耦，二者只通过消息发生联系，对象之间彼此都没有关联，大大降低了对象间的耦合性。下面是一个典型的消息总线使用示例：

```
void test_messge_bus()
{
message_bus bus;
//消息接受者向消息总线注册函数
bus.register_handler("test", [](int
a, int b) { return a + b; });
bus.register_handler("void", [] {
std::cout << "void" << std::endl; });
bus.register_handler("str", []
(int a, const std::string& b) { return
std::to_string(a) + b; });
//调用者通过消息总线发起调用
auto r = bus.call<int>("test", 1,
2);
std::cout << r << std::endl;
auto s = bus.
call<std::string>("str", 1,
std::string("test"));
bus.call_void("void");
}
```

消息总线可以注册任意函数，包括带参数和不带参数的，以及有返回类型和没有返回类型的函数，使用很方便。调用也很简单，传入消息 key 和调用参数即可。这个消息总线彻底解耦了调用者和被调用者。之前的示例中并没有解决获取调用返回值的问题，解决起来也很容易，将需要返回的结果作为一个 void* 指针传入到 apply 中赋值既可。下面是 message\_bus 的具体实现：

```
#include <string>
#include <map>
#include <functional>
#include "function_traits.hpp"
class message_bus
{
public:
template<typename Function>技术 Technology
112
void register_handler(std::string
const & name, const Function& f)
{
using std::placeholders::_1;
using std::placeholders::_2;

this->invokers_[name] = { std::
bind(&invoker<Function>::apply, f, _1,
_2) };
}

template <typename T, typename ...
Args>
T call(const std::string& name,
Args&& ... args)
{
auto it = invokers_.find(name);
if (it == invokers_.end())
return{};
auto args_tuple = std::make_tupl
e(std::forward<Args>(args)...);

char data[sizeof(std::tuple<Ar
gs...>)];
std::tuple<Args...>* tp = new
(data) std::tuple<Args...>;
*tp = args_tuple;
T t;
it->second(tp, &t);
return t;
}
private:
template<typename Function>
struct invoker
{
static inline void apply(const
Function& func, void* bl, void* result)
{
using tuple_type = typename
timax::function_traits<Function>::tuple_
type;
const tuple_type* tp =
static_cast<tuple_type*>(bl);

call(func, *tp, result);
}
template<typename F, typename
... Args>
static typename std::enable_
if<std::is_void<typename std::result_of<
F(Args...)>::type>::value>::type
call(const F& f, const
std::tuple<Args...>& tp, void*)
{
call_helper(f, std::make_
index_sequence<sizeof... (Args)>{}, tp);
}
template<typename F, typename
... Args>
static typename std::enable_
if<!std::is_void<typename std::result_of
<F(Args...)>::type>::value>::type
call(const F& f, const
std::tuple<Args...>& tp, void* result)
{
auto r = call_helper(f,
std::make_index_sequence<sizeof...
(Args)>{}, tp);
*(decltype(r)*)result = r;
}
template<typename F, size_t...
I, typename ... Args>
static decltype(auto)
call_helper(const F& f, const
std::index_sequence<I...>&, const
std::tuple<Args...>& tup)
{
return
f(std::get<I>(tup)...);
}
};
private:
std::map<std::string,
std::function<void(void*, void*)>>
invokers_;
};
```

需要注意的是，为了简单起见，没有考虑异常和线程安全。如果调用参数和实际参数不符合，还原函数调用的时候会抛异常，一个办法是做异常处理，另外一个办法是通过调用协在编译期检查调用是否正确。

### 总结

本文通过 C++ 11/14的一些新特性和模版元技巧解决了一个技术难题：如何实现通用的容器保存所有类型函数并实现调用。这个难题解决之后可以利用该技术实现灵活易用的消息总线，它彻底解耦了调用者和被调用者，体现了现代 C++ 的强大和灵活。
