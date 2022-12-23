## 现代 C++ 函数式编程

文/祁宇

>本文作者从介绍函数式编程的概念入手，分析了函数式编程的表现形式和特性，最终通过现代 C++ 的新特性以及一些模板云技巧实现了一个非常灵活的pipeline，展示了现代 C++ 实现函数式编程的方法和技巧，同时也体现了现代 C++ 的强大威力和无限可能。

### 概述

函数式编程是一种编程范式，它有下面的一些特征：

- 函数是一等公民，可以像数据一样传来传去

- 高阶函数

- 递归

- pipeline

- 惰性求值

- 柯里化

- 偏应用函数

C++ 98/03中的函数对象，与 C++ 11中的Lambda表达式、std::function 以及 std::bind 让 C++ 的函数式编程变得容易。我们可以利用 C++ 11/14里的新特性来实现高阶函数、链式调用、惰性求值和柯里化等函数式编程特性。本文将通过一些典型示例来讲解如何使用现代 C++ 实现函数式编程。

### 高阶函数和 pipeline 的表现形式

高阶函数就是参数为函数或返回值为函数的函数，经典的高阶函数包括 map、filter、fold 和 compose 函数，比如 Scala 中高阶函数：

- map

```
numbers.map((i: Int) => i * 2)
```

    对列表中的每个元素应用一个函数，返回应用后的元素所组成的列表。

- filter

```
numbers.filter((i: Int) => i % 2 == 0)
```

    移除任何对传入函数计算结果为 false 的元素。

- fold

```
numbers.fold(0) { (z, i) =>
  a + i
}
```

    将一个初始值和一个二元函数的结果累加起来。

- compose

```
val fComposeG = f _ compose g _
fComposeG("x")
```

    组合其它函数形成一个新函数 f(g(x))。

上面的例子中，有的是参数为函数，有的是参数和返回值都是函数。高阶函数不仅语义上更加抽象泛化，还能实现“函数是一等公民”，将函数像 data 一样传来传去或组合，非常灵活。其中，compose 还可以实现惰性求值，其的返回结果是一个函数，我们可以保存起来，在后面需要的时候调用。

pipeline 把一组函数放到一个数组或是列表中，然后把数据传给这个列表。数据就像一个链条一样顺序地被各个函数所操作，最终得到我们想要的结果。它的设计哲学就是让每个功能只做一件事，并将其做到极致，软件或程序的拼装会变得更为简单和直观。

Scala 中的链式调用是这样的：

```
s(x) = (1 to x) |> filter (x => x % 2 == 0) |> map (x => x * 2)
```

用法和 Unix Shell 的管道操作比较像，|前面的数据或函数作为|后面函数的输入，顺序执行直到最后一个函数。

这种管道方式的函数调用让逻辑看起来更加清晰明了，也非常灵活，允许你将多个高阶函数自由组合成一个链条，同时还可以保存起来实现惰性求值。现代 C++ 实现这种 pipeline 也是比较容易的，接下来讲解如何充分借助 C++ 11/14的新特性来实现这些高阶函数和 pipeline。

### 实现 pipeline 的关键技术

根据前面介绍的 pipeline 表现形式，可以把 pipeline 分解为几部分：高阶函数、惰性求值、运算符|、柯里化和 pipeline，把这几部分实现之后就可以组成一个完整的 pipeline 了。下面来分别介绍它们的实现技术。

#### 高阶函数

函数式编程的核心就是“函数”，它是一等公民，最灵活的函数就是高阶函数，现代 C++ 的算法中已经有很多高阶函数了，比如 for\_each, transform：

```
std::vector<int> vec{1,2,3,4,5,6,7,8,9}
//接受一个打印的Lambda表达式
std::for_each(vec.begin(), vec.end(), []
(auto i){ std::cout<<i<<std::endl; });
//接受一个转换的Lambda表达式
transform(vec.begin(), vec.end(), vec.
begin(), [](int i){ return i*i; }); 
```

这些高阶函数不仅可以接受 Lambda 表达式，还能接受 std::function、函数对象、普通的全局函数，很灵活。需要注意的是，普通的全局函数在 pipeline 时存在局限性，因为它不像函数对象一样可以保存起来延迟调用，所以我们需要一个方法将普通的函数转换为函数对象。std::bind 也可以将函数转化为函数对象，但是 bind不够通用，使用时它只能绑定有限的参数，如果函数本身就是可变参数的就无法 bind 了。所以，这个函数对象必须是泛化的，类似于这样：

```
class universal_functor
{
public:
template <typename... Args>
auto operator()(Args&&... args) const
->decltype(globle_func(std::forward<Args
>(args)...))
{
return globle_func(std::f
orward<Args>(args)...);
}
};
```

上面的函数对象内部包装了一个普通函数的调用，当函数调用时，实际上会调用普通函数 globle_func，但是这个代码不通用，无法用于其他函数。为了让这个转换变得通用，我们可以借助一个宏来实现 function 到 functor 的转换。

```
#define define_functor_type(func_name)
class tfn_##func_name {\
public: template <typename... Args>
auto operator()(Args&&... args) const
->decltype(func_name(std::forward<Args>(
args)...))\
{ return func_name(std::forward<Args>(ar
gs)...); } }

//test code
int add(int a, int b)
{
return a + b;
}
int add_one(int a)
{
return 1 + a;
}

define_functor_type(add);
define_functor_type(add_one);

int main()
{
tnf_add add_functor;
add_functor(1, 2); //result is 3
tfn_add_one add_one_functor;
add_one_functor(1); //result is 2

return 0;
}
```

我们先定义了一个宏，这个宏根据参数来生成一个可变参数的函数对象，其类型名为 tfn\_ 加普通函数的函数名，之所以要加一个前缀tfn\_，是为了避免类型名和函数名重名。define\_functor\_type 宏只是定义了一个函数对象的类型，用起来略感不便，还可以再简化一下，让使用更方便。我们可以再定义一个宏来生成转换后的函数对象：

```
#define make_globle_functor(NAME, F) const auto NAME = define_functor_type(F); 
//test code
make_globle_functor(fn_add, add);
make_globle_functor(fn_add_one, add_one);
 
int main()
{
    fn_add(1, 2);
    fn_add_one(1);
 
    return 0;   
}
```

make\_globle\_functor 生成了一个可以直接使用的全局函数对象，使用起来更方便了。用这个方法就可以将普通函数转成 pipeline 中的函数对象了。接下来我们来探讨实现惰性求值的关键技术。

#### 惰性求值

惰性求值是将求值运算延迟到需要值时候进行，通常的做法是将函数或函数的参数保存起来，在需要的时候才调用函数或者将保存的参数传入函数实现调用。现代 C++ 里已经提供可以保存起来的函数对象和 Lambda 表达式，因此需要解决的问题是如何将参数保存起来，然后在需要时传给函数实现调用。我们可以借助 std::tuple、type\_traits 和可变模版参数来实现目标。

```
template<typename F, size_t... I,
typename ... Args>
inline auto tuple_apply_impl(const F& f,
const std::index_sequence<I...>&, const
std::tuple<Args...>& tp)
{
return f(std::get<I>(tp)...);
}
template<typename F, typename ... Args>
inline auto tuple_apply(const F& f,
const std::tuple<Args...>& tp) -> declty
pe(f(std::declval<Args>()...))
{
return tuple_apply_impl(f,
std::make_index_sequence<sizeof...
(Args)>{}, tp);
}
int main()
{
//test code
auto f = [](int x, int y, int z) {
return x + y - z; };
//将函数调用需要的参数保存到tuple中
auto params = make_tuple(1, 2, 3);
//将保存的参数传给函数f， 实现函数调用
auto result = tuple_apply(f, params); //
result is 0

return 0;
}
```

上面的测试代码中，我们先把参数保存到一个 tuple 中，然后在需要的时候将参数和函数 f 传入 tuple\_apply，最终实现了 f 函数的调用。tuple\_apply 实现了一个“魔法”将 tuple 变成了函数的参数，来看看这个“魔法”具体是怎么实现的。

tuple\_apply\_impl 实现的关键是在于可变模版参数的展开，可变模版参数的展开又借助了 std::index_sequence<I...>。这个 sequence 是一个 size\_t 的序列，由 std::make\_index_sequence<sizeof... (Args)>{} 生成，它会根据参数的个数来生成这个序列，比如这个例子中有三个参数，所以它会生成 std::index_sequence<1,2,3> 序列，f(std::get＜I＞(tp)...)则会展开这个序列，分别调用 std::get<0>(tp), std::get<1>(tp), std::get<2>(tp)，取出来的三个参数就会作为函数的入参，从而实现调用。

这样，我们就实现了惰性求值的一个关键技术：将参数保存在 tuple 中实现延迟调用。除此之外，我们还可以将函数保存在 tuple 中实现一个函数链，将在后面介绍。

#### 运算符 operator|

pipeline 的一个主要表现形式是通过运算符|来将 data 和函数分隔开，或将函数和函数组成一个链条，比如像下面的 Unix Shell 命令：

```
ps auwwx | awk '{print $2}' | sort -n |
xargs echo
template<typename T, class F>
auto operator|(T&& param, const F& f) ->
decltype(f(std::forward<T>(param)))
{
return f(std::forward<T>(param));
}
//test code
auto add_one = [](auto a) { return 1 +
a; };
auto result = 2 | add_one; //result is 3>
```

除了 data 和函数通过|连接之外，还需要实现函数和函数通过|连接，我们通过可变参数来实现：

```
template<typename... FNs, typename F>
inline auto operator|(fn_chain<FNs...>
&& chain, F&& f)
{
return chain.
add(std::forward<F>(f));
}

//test code
auto chain = fn_chain<>() | (filter >>
[](auto i) { return i % 2 == 0; }) |
ucount | uprint;
```

其中 fn\_chain 是一个可以接受任意函数的函数对象，它的实现将在后面介绍。通过|运算符重载可以实现类似于 Unix Shell 的 pipeline 表现形式。

#### 柯里化

函数式编程中比较灵活的一个地方就是柯里化（currying），柯里化是把多个参数的函数变换成单参数的函数，并返回一个新函数，由它处理剩下的参数。以 Scala 的柯里化为例：

- 未柯里化的函数

```
def add(x:Int, y:Int) = x + y
add(1, 2)   // 3
add(7, 3)   // 10
```

- 柯里化之后

```
def add(x:Int) = (y:Int) => x + y
add(1)(2)   // 3
add(7)(3)   // 10
```

Currying 之后 add(1)(2) 等价于 add(1,2)，这种 Currying 默认是从左到右的，但大部分编程语言没有实现更灵活的 Curring。C++ 11里面的 std::bind 可以实现 Currying，但要实现向左或向右灵活的 Currying 比较困难，可以借助 tuple 和前面介绍的 tuple_apply 来实现一个更灵活的 Currying 函数对象。

```
template<typename F, typename Before
= std::tuple<>, typename After =
std::tuple<>>
class curry_functor {
private:
F f_; ///< main
functor
Before before_; ///< curryed
arguments
After after_; ///< curryed
arguments
public:
curry_functor(F && f) : f_
(std::forward<F>(f)), before_
(std::tuple<>()), after_(std::tuple<>())
{}

curry_functor(const F & f, const
Before & before, const After & after) :
f_(f), before_(before), after_(after) {}

template <typename... Args>
auto operator()(Args... args)
const -> decltype(tuple_apply(f_,
std::tuple_cat(before_, make_
tuple(args...), after_)))
{
// execute via tuple
return tuple_apply(f_,
std::tuple_cat(before_, make_tuple(std::
forward<Args>(args)...), after_));
}
// currying from left to right
template <typename T>
auto curry_before(T && param)
const
{
using RealBefore =
decltype(std::tuple_cat(before_,
std::make_tuple(param)));
return curry_
functor<F, RealBefore, After>(f_,
std::tuple_cat(before_, std::make_
tuple(std::forward<T>(param))), after_);
}
// currying from righ to left
template <typename T>
auto curry_after(T && param)
const
{
using RealAfter =
decltype(std::tuple_cat(after_,
std::make_tuple(param)));
return curry_functor<F,
Before, RealAfter>(f_, before_,
std::tuple_cat(after_, std::make_tuple(s
td::forward<T>(param))));
}
};

template <typename F>
auto fn_to_curry_functor(F && f)
{
return curry_
functor<F>(std::forward<F>(f));
}

//test code
void test_count()
{
auto f = [](int x, int y, int z)
{ return x + y - z; };
auto fn = fn_to_curry_functor(f);

auto result = fn.curry_before(1)
(2, 3); //0
result = fn.curry_before(1).
curry_before(2)(3); //0
result = fn.curry_before(1).
curry_before(2).curry_before(3)(); //0

result = fn.curry_before(1).
curry_after(2).curry_before(3)(); //2
result = fn.curry_after(1).curry_
after(2).curry_before(2)(); //1
}
```

从测试代码中可以看到这个 Currying 函数对象，既可以从左边 Currying 又可以从右边，非常灵活。不过使用上还不太方便。我们可以通过运算符重载来简化书写，由于 C++ 标准中不允许重载全局的 operater() 符，并且 operater() 符无法区分到底是从左边还是从右边 Currying，所以选择重载<<和>>操作符来分别表示从左至右 Currying 和从右至左 Currying。

```
// currying from left to right
template<typename UF, typename Arg>
auto operator<<(const UF & f, Arg &&
arg) -> decltype(f.template curry_before
<Arg>(std::forward<Arg>(arg)))
{
return f.template curry_before<Ar
g>(std::forward<Arg>(arg));
}

// currying from right to left
template<typename UF, typename Arg>
auto operator>>(const UF & f, Arg &&
arg) -> decltype(f.template curry_after<
Arg>(std::forward<Arg>(arg)))
{
return f.template curry_after<Arg
>(std::forward<Arg>(arg));
}
```

有了这两个重载运算符，测试代码可以写得更简洁了。

```
void test_currying()
{
    auto f = [](int x, int y, int z) { return x + y - z; };
    auto fn = fn_to_curry_functor(f);
 
auto result = (fn << 1)(2, 3); //0
    result = (fn << 1 << 2)(3); //0
result = (fn << 1 << 2 << 3)(); //0
result = (fn << 1 >> 2 << 3)(); //2
result = (fn >> 1 >> 2 << 3)(); //1
}
```

curry\_functor 利用了 tuple 的特性，内部有两个空的 tuple，一个用来保存 left currying 的参数，一个用来保存 right currying 的参数，不断地 Currying 时，通过 tuple\_cat 把新 Currying 的参数保存到 tuple 中，最后调用时将 tuple 成员和参数组成一个最终的 tuple，然后通过 tuple\_apply 实现调用。有了前面这些基础设施之后我们实现 pipeline 也是水到渠成。

#### pipeline

通过运算符|重载，可以实现一个简单的 pipeline：

```
template<typename T, class F>
auto operator|(T&& param, const F& f) ->
decltype(f(std::forward<T>(param)))
{
return f(std::forward<T>(param));
}
//test code
void test_pipe()
{
auto f1 = [](int x) { return x + 3; };
auto f2 = [](int x) { return x *
2; };
auto f3 = [](int x) { return
(double)x / 2.0; };
auto f4 = [](double x) {
std::stringstream ss; ss << x; return
ss.str(); };
auto f5 = [](string s) { return
"Result: " + s; };
auto result = 2|f1|f2|f3|f4|f5;
//Result: 5
}
```

这个简单的 pipeline 虽然可以实现管道方式的链式计算，但是它只是将 data 和函数通过|连接起来了，还没有实现函数和函数的连接，并且是立即计算的，没有实现延迟计算。因此还需要通过|连接函数，从而实现灵活的 pipeline。我们可以通过一个 function chain 来接受任意个函数并组成一个函数链。利用可变模版参数、tuple 和 type_traits 就可以实现了。

```
template <typename... FNs>
class fn_chain {
private:
const std::tuple<FNs...>
functions_;
const static size_t TUPLE_SIZE =
sizeof...(FNs);

template<typename Arg, std::size_
t I>
auto call_impl(Arg&& arg,
const std::index_sequence<I>&) const
->decltype(std::get<I>(functions_)
(std::forward<Arg>(arg)))
{
return
std::get<I>(functions_)
(std::forward<Arg>(arg));
}

template<typename Arg, std::size_
t I, std::size_t... Is>
auto call_impl(Arg&& arg,
const std::index_sequence<I,
Is...>&) const ->decltype(call_
impl(std::get<I>(functions_)
(std::forward<Arg>(arg)), std::index_
sequence<Is...>{}))
{
return call_
impl(std::get<I>(functions_)
(std::forward<Arg>(arg)), std::index_
sequence<Is...>{});
}

template<typename Arg>
auto call(Arg&& arg)
const-> decltype(call_
impl(std::forward<Arg>(arg), std::make_
index_sequence<sizeof...(FNs)>{}))
{
return call_
impl(std::forward<Arg>(arg), std::make_
index_sequence<sizeof...(FNs)>{});
}

public:
fn_chain() : functions_
(std::tuple<>()) {}
fn_chain(std::tuple<FNs...>
functions) : functions_(functions) {}

// add function into chain
template< typename F >
inline auto add(const F& f) const
{
return fn_chain<FNs...,
F>(std::tuple_cat(functions_, std::make_
tuple(f)));
}

// call whole functional chain
template <typename Arg>
inline auto operator()(Arg&& arg)
const -> decltype(call(std::forward<Arg>
(arg)))
{
return
call(std::forward<Arg>(arg));
}
};

// pipe function into functional chain
via | operator
template<typename... FNs, typename F>
inline auto operator|(fn_chain<FNs...>
&& chain, F&& f)
{
return chain.
add(std::forward<F>(f));
}

#define tfn_chain fn_chain<>()

//test code
void test_pipe()
{
auto f1 = [](int x) { return x + 3; };
auto f2 = [](int x) { return x *
2; };
auto f3 = [](int x) { return
(double)x / 2.0; };
auto f4 = [](double x) {
std::stringstream ss; ss << x; return
ss.str(); };
auto f5 = [](string s) { return
"Result: " + s; };
auto compose_fn = tfn_
chain|f1|f2|f3|f4|f5; //compose a chain
compose_fn(2); // Result: 5
}
```

测试代码中用一个 fn\_chain 和运算符|将所有的函数组合成了一个函数链，在需要的时候调用，从而实现了惰性求值。

fn\_chain 的实现思路是这样的：内部有一个 std::tuple<FNs...> 的成员变量，用来保存新加入到链条中的函数，当调用|运算符的时候，就会将|后面的函数添加到内部的 tuple 成员变量中，并返回一个新的 fn\_chain 函数对象，用来实现惰性求值。在真正调用 fn\_chain 时，call 函数会将 tuple 展开，然后依次调用 tuple 中的函数，前一个函数的输出作为后一个函数的输入，最终完成整个函数链的调用。展开 tuple 是通过 std::index\_sequence 来展开的，关键在这两个函数：

```
template<typename Arg, std::size_t I>
auto call_impl(Arg&& arg,
const std::index_sequence<I>&) const
->decltype(std::get<I>(functions_)
(std::forward<Arg>(arg)))
{
return
std::get<I>(functions_)
(std::forward<Arg>(arg));
}

template<typename Arg, std::size_
t I, std::size_t... Is>
auto call_impl(Arg&& arg,
const std::index_sequence<I,
Is...>&) const ->decltype(call_
impl(std::get<I>(functions_)
(std::forward<Arg>(arg)), std::index_
sequence<Is...>{}))
{
return call_
impl(std::get<I>(functions_)
(std::forward<Arg>(arg)), std::index_
sequence<Is...>{});
}
```

在调用 call\_impl 的过程中，将 std::index\_sequence 不断展开，先从 tuple 中获取第I个function，然后调用获得第I个 function 的执行结果，将这个执行结果作为下次调用的参数，不断地递归调用，直到最后一个函数完成调用为止，返回最终的链式调用的结果。

至此我们实现具备惰性求值、高阶函数和 Currying 特性的完整的 pipeline。有了这个 pipeline，我们可以实现经典的流式计算和 AOP，接下来我们来看看如何利用 pipeline 来实现流式的 MapReduce 和灵活的 AOP。

### 实现一个 pipeline 形式的 mapreduce 和 AOP

前面的 pipeline 已经可以实现链式调用了，要实现 pipeline 形式的 MapReduce 关键就是实现 map、filter 和 reduce 等高阶函数。下面是它们的具体实现：

```
// MAP
template <typename T, typename... TArgs,
template <typename...>class C, typename
F>
auto fn_map(const C<T, TArgs...>&
container, const F& f) -> C<decltype(f(s
td::declval<T>()))>
{
using resultType =
decltype(f(std::declval<T>()));
C<resultType> result;
for (const auto& item :
container)
result.push_
back(f(item));
return result;
}

// REDUCE (FOLD)
template <typename TResult, typename
T, typename... TArgs, template
<typename...>class C, typename F>
TResult fn_reduce(const C<T, TArgs...>&
container, const TResult& startValue,
const F& f)
{
TResult result = startValue;
for (const auto& item :
container)
result = f(result, item);
return result;
}

// FILTER
template <typename T, typename... TArgs,
template <typename...>class C, typename
F>
C<T, TArgs...> fn_filter(const C<T,
TArgs...>& container, const F& f)
{
C<T, TArgs...> result;
for (const auto& item :
container)
if (f(item))
result.push_
back(item);
return result;
}
```

这些高阶函数还需要转换成支持 Currying 的 Functor，前面我们已经定义了一个普通的函数对象转换为柯里化的函数对象的方法：

```
template <typename F>
auto fn_to_curry_functor(F && f)
{
return curry_
functor<F>(std::forward<F>(f));
}
```

通过下面这个宏让 currying functor 用起来更简洁：

```
#define make_globle_curry_functor(NAME, F) define_functor_type(F); const auto NAME = fn_to_curry_functor(tfn_##F());
 
make_globle_curry_functor(map, fn_map);
make_globle_curry_functor(reduce, fn_reduce);
make_globle_curry_functor(filter, fn_filter);
```

我们定义了 map、reduce 和 filter 支持柯里化的三个全局函数对象，接下来就可以把它们组成一个 pipeline 了。

```
void test_pipe()
{
//test map reduce
vector<string> slist = { "one",
"two", "three" };

slist | (map >> [](auto s) {
return s.size(); })
| (reduce >> 0 >> [](auto
a, auto b) { return a + b; })
| [](auto a) { cout << a
<< endl; };

//test chain, lazy eval
auto chain = tfn_chain | (map >>
[](auto s) { return s.size(); })
| (reduce >> 0 >> [](auto
a, auto b) { return a + b; })
| ([](int a) { std::cout
<< a << std::endl; });

slist | chain;
}
```

上面的例子实现了 pipeline 的 MapReduce，这个 pipeline 支持 currying 还可以任意组合，非常方便和灵活。

有了这个 pipeline，实现灵活的 AOP 也是很容易的

```
struct person
{
    person get_person_by_id(int id)
    {
        this->id = id;
        return *this;
    }
 
    int id;
    std::string name;
};
void test_aop()
{
const person& p = { 20, "tom" };
    auto func = std::bind(&person::get_person_by_id, &p, std::placeholders::_1); 
    auto aspect = tfn_chain | ([](int id) { cout << "before"; return id + 1; })
        | func
        | ([](const person& p) { cout << "after" << endl; });
 
    aspect(1);
}
```

上面的测试例子中，核心逻辑是 func 函数，我们可以在 func 之前或之后插入切面逻辑，切面逻辑可以不断地加到链条前面或者后面，实现很巧妙，使用很常灵活。

### 总结

本文从介绍函数式编程的概念入手，分析了函数式编程的表现形式和特性，最终通过现代 C++ 的新特性和一些模版元技巧实现了一个非常灵活的 pipeline，展示了现代 C++ 实现函数式编程的方法和技巧，同时也体现了现代 C++ 的强大威力和无限可能。文中完整的代码可以从 https://github.com/qicosmos/cosmos/blob/master/modern_functor.hpp 上查看。

本文的代码和思路参考和借鉴了 http://vitiy.info/templates-as-first-class-citizens-in-cpp11/，在此表示感谢。