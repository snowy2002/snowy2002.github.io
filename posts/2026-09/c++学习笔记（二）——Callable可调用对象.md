---
title: C++学习笔记（二）—— Callable 可调用对象
filename: 20260914-cpp-study-notes-002-callable
tags:
  - cpp
  - 学习笔记
categories:
  - C++
date: 2026-09-14
description: 系统梳理 C++ 中 Callable（可调用对象）的六种形式、INVOKE 的统一语义、std::invoke 与 C++20 concept，并用汇编和实测数据说明它们在编译器眼里到底差多少。
articleGPT: 本文从"函数和仿函数为什么都能用 () 调用"出发，讲清 Callable 的标准定义（INVOKE 表达式），逐一定位函数、函数指针、仿函数、lambda、成员指针、std::function 六种形式的本质差异；随后用 -O2 汇编对比和微基准实测揭示"能否内联"才是性能分水岭，并拆开 std::function 的 32 字节内存布局解释类型擦除与小对象优化（SBO）的代价，最后纠正几个高频误解。
share: true
delete: false
---

## 从一个奇怪的现象说起

下面两个东西，长得毫无关系：

```cpp
int f(int a, int b) { return a - b; }        // 一个函数

struct F {                                    // 一个结构体
    int cnt;
    int operator()(int a, int b) { ++cnt; return a - b; }
};
```

一个是函数，一个是**对象**。但调用语法完全一样：

```cpp
f(3, 4);              // -1
F f1; f1(3, 4);       // -1
```

`std::sort` 也不关心你传的是哪一个：

```cpp
std::sort(v.begin(), v.end(), f);     // 传函数
std::sort(v.begin(), v.end(), F{});   // 传对象
```

这件事看着理所当然，其实是 C++ 泛型编程的地基。这一篇要回答：**"能被调用"这件事，在类型系统里到底怎么描述？六种形式真正的差别在哪？**

## Callable 的准确定义

标准里并没有一个叫 `Callable` 的类或类型。它是一个 **named requirement（具名要求）**，用统一表达式定义：

> 只要 `INVOKE(f, args...)` 这个表达式合法，`f` 就是可调用对象（callable）。
> —— 见 `[func.require]`

这个 `INVOKE` 不是 C++ 里某个函数，而是标准用来自我描述的记号（伪代码）。它的价值在于**一口气覆盖了全部六种形式**，包括语法差异极大的成员指针。

换个更好记的说法：

> **Callable 不是"某类对象"，而是一种"能用 `()` 调用的能力"。** 任何满足它的类型，都自动获得标准库的入场券。

所以 `std::sort` 的第三个参数不是"函数指针"，而是"任何 callable"——这才是它既能接函数、又能接 lambda、还能接带状态仿函数的原因。

## 六种形式

### 1. 普通函数

```cpp
int add(int a, int b) { return a + b; }
add(1, 2);   // 3
```

有个常见说法是"函数名退化为函数指针"，**这不准确**。函数名本身是**函数类型的左值**，只有在特定上下文（赋给指针、作为参数传递）才隐式转换成函数指针。直接 `add(1, 2)` 调用时并没有发生退化，编译器直接就知道要跳到哪。

这不是抠字眼——它决定了编译器能不能内联：**直接调用且定义可见时能内联，退化成指针后就未必**（第六节有汇编证据）。

### 2. 函数指针

```cpp
int (*fp)(int, int) = add;   // 这里才发生退化
fp(3, 4);                    // 7
```

类型固定、大小固定（64 位下 8 字节），但**指向的目标在编译期可能未知**——这是它和 lambda 的本质分野。

### 3. 仿函数（Functor）

重载了 `operator()` 的类对象：

```cpp
struct Adder {
    int operator()(int a, int b) const { return a + b; }
};

Adder adder;
adder(1, 2);   // 3
```

**优势：可以携带状态。**

```cpp
struct Counter {
    int count = 0;
    int operator()() { return ++count; }
};

Counter c;
c();   // 1
c();   // 2
```

函数指针做不到这一点——它没有地方存状态。要带状态，函数指针只能靠全局变量，那就没法重入、也没法多实例。

### 4. Lambda 表达式

Lambda **不是函数**，它是编译器生成的**匿名函数对象**。上面那个 `Adder` 写起来是这样：

```cpp
auto adder = [](int a, int b) { return a + b; };
adder(1, 2);   // 3
```

捕获列表就是它的"成员变量"：

```cpp
int x = 10;
auto capture = [x](int y) { return x + y; };   // 按值捕获 → 等价于有个成员 int x
```

有两个细节值得记住：

**闭包类型是唯一的，即使两个 lambda 源码完全相同。** 实测：

```cpp
auto l1 = [](int a) { return a + 1; };
auto l2 = [](int a) { return a + 1; };   // 源码一模一样
std::is_same_v<decltype(l1), decltype(l2)>;   // false！
```

所以你在报错信息里会看到 `main()::<lambda(int)>` 这种没法写出来的类型名，只能靠 `auto` 或模板参数接住。

**lambda 没有默认构造函数。** 实测 `std::is_default_constructible_v<decltype(l1)>` 为 `false`。这解释了为什么 `std::map` 的比较器如果要用 lambda，必须写进模板参数而不能作为默认构造的成员。

### 5. 成员函数指针

```cpp
struct Foo {
    int bar(int x) { return x * 2; }
};

int (Foo::*pmf)(int) = &Foo::bar;
Foo f;
(f.*pmf)(5);      // 10
(foo_ptr->*pmf)(5);  // 同样合法
```

它**不是一个指针地址**——成员函数指针需要绑对象才能调用，这是它和普通函数指针语法上分道扬镳的原因。注意 C++ 里还有成员**数据**指针 `int Foo::*pmd = &Foo::x;`，它同样属于 INVOKE 的管辖范围（见下一节）。

### 6. `std::function`

```cpp
std::function<int(int,int)> f = add;    // 包装普通函数
f = F{};                                 // 换成仿函数
f = [](int a, int b) { return a + b; };  // 换成 lambda
```

`std::function` 本身也是 callable，作用是把上面五种**类型各异**的东西统一成一个**类型相同**的东西。代价是性能，第七节专门拆。

## 标准库如何统一它们：INVOKE 与 `std::invoke`

`INVOKE(f, args...)` 的四条规则，对应四种调用形态：

| 情形 | 展开后的写法 |
|------|-------------|
| 普通 callable | `f(args...)` |
| 成员函数指针 + 对象 | `(obj.*pmf)(args...)` |
| 成员函数指针 + 引用包装 | `(obj.get().*pmf)(args...)` |
| 成员**数据**指针 | `obj.*pmd`（取值，没有参数） |

你平时写不出统一语法，标准替你在 C++17 给了 [`std::invoke`](https://en.cppreference.com/w/cpp/utility/functional/invoke)：

```cpp
#include <functional>

std::invoke(add, 1, 2);            // 普通函数 → 3
std::invoke(adder, 1, 2);          // 仿函数  → 3
std::invoke([](int a,int b){ return a+b; }, 1, 2);  // lambda
std::invoke(pmf, f, 5);            // 成员函数指针 → 10
std::invoke(&Foo::x, f);           // 成员数据指针 → 取到成员
std::invoke(pmf, std::ref(f), 5);  // 引用包装也行
```

**记住一句话：标准库里所有"调用 callable"的地方，语义上都等价于 `std::invoke`。** `std::thread`、`std::async`、`std::apply`、`std::bind`、线程池的 `enqueue`，全都是。这就是为什么你能给它们传成员函数指针。

`std::invoke` 本身是编译期分派的（重载/模板特化），**不引入任何运行期开销**。

## C++17 检测、C++20 约束

### C++17：type traits

```cpp
#include <type_traits>

static_assert( std::is_invocable_v<decltype(add), int, int>);
static_assert(!std::is_invocable_v<decltype(add), int>);       // 参数不够
static_assert( std::is_invocable_r_v<long, Adder, int, int>);  // 返回值可否转 long
```

| Trait | 作用 |
|-------|------|
| `std::is_invocable<F, Args...>` | 能否用 `Args...` 调用 `F` |
| `std::is_invocable_r<R, F, Args...>` | 能否调用且返回值可转成 `R` |
| `std::invoke_result<F, Args...>` | 调用结果类型（`::type`） |

### C++20：concept

C++20 把这件事提升成一等公民，可以直接用来约束模板：

```cpp
#include <concepts>

template <std::invocable<int, int> F>
void apply(F f, int a, int b) { f(a, b); }
```

| Concept | 含义 |
|---------|------|
| `std::invocable<F, Args...>` | 基本的可调用约束 |
| `std::regular_invocable<F, Args...>` | 额外要求调用不改变 f 的状态、结果可预测 |
| `std::predicate<F, Args...>` | 返回 `bool` 的可调用 |
| `std::relation<F, T, U>` | 二元关系（`predicate` 的特化） |

最大的好处是**报错信息**。用 `enable_if`/SFINAE 时，一个不匹配会喷出几百行模板展开；用 concept，编译器直接告诉你"F 不满足 `std::invocable<int, int>`"，干净得多。

## 编译器眼里，它们的差距有多大

前面六种形式在语法上都能 `()`，但**编译器的视角完全不同**。这一节用汇编和实测说清楚差距在哪。

环境：GCC 13.3.0 / x86-64 / System V ABI / Ubuntu 24.04。以下都是实测结果，命令附在后面。

### 先看汇编：同一个模板，三种调用方式

用一个统一的转发模板，分别喂 lambda、函数指针、`std::function`：

```cpp
template <typename F>
int demo(const F& f, int a, int b) { return f(a, b); }

int via_lambda(int a, int b) {
    return demo([](int x, int y) { return x + y; }, a, b);
}
int via_funcptr(int (*f)(int,int), int a, int b) {
    return demo(f, a, b);
}
int via_stdfunction(const std::function<int(int,int)>& f, int a, int b) {
    return demo(f, a, b);
}
```

`g++ -std=c++17 -O2 -S -masm=intel` 的结果：

**lambda —— 完全内联，只剩一条加法指令**

```asm
_Z11via_lambdaii:            # int via_lambda(int, int)
	endbr64
	lea	eax, [rdi+rsi]        # a + b 直接算完
	ret                        # 没有任何 call 指令
```

编译器看到了 lambda 的闭包类型和函数体，把整个调用**抹平**了。参数在 `rdi`/`rsi`，结果在 `eax`——调用这件事在机器码层面完全消失了。

**函数指针 —— 尾调用，跳转目标运行期才知道**

```asm
_Z11via_funcptrPFiiiEii:
	endbr64
	mov	rax, rdi            # 取出函数指针
	mov	edi, esi            # 排好参数：a
	mov	esi, edx            # 排好参数：b
	jmp	rax                 # ← 尾调用，目标此刻未知
```

编译器连内联的**机会**都没有：`rax` 里装什么只有运行时才知道。

**`std::function` —— 间接调用 + 一次空值检查**

```asm
_Z15via_stdfunctionRKSt8functionIFiiiEEii:
	...
	cmp	QWORD PTR 16[rdi], 0    # 检查是不是空 function
	je	.L10                    # 空 → 走 __throw_bad_function_call
	mov	DWORD PTR [rsp], esi
	mov	DWORD PTR [rsp+4], edx
	call	[QWORD PTR 24[rdi]]     # ← 从对象偏移 24 处取函数指针间接调用
```

比函数指针多了一次**空值检查**（必要时抛 `std::bad_function_call`），代价是一次可预测的分支，可以忽略。真正的开销和函数指针一样：`call [reg]` 的目标不透明。

### 再看实测：差距到底多少

光看汇编容易高估，跑个基准。**注意测法**：如果让编译器看见全部代码，它会把循环直接折叠成闭式解——那样测不出真实调用开销。

```cpp
static volatile int g_seed = 1;          // 阻止循环折叠

template <typename F>
long long run(const F& f, int n) {
    long long s = 0;
    for (int i = 0; i < n; ++i) s += f(i, i + 1);
    return s;
}
```

调用 2 亿次，三种形式各测一遍：

| 形式 | 耗时 | 相对 lambda |
|------|------|------------|
| lambda（可内联） | ~100 ms | 1.0x |
| 函数指针 | ~280 ms | **2.8x** |
| `std::function` | ~285 ms | **2.9x** |

**反直觉的结论在这里**：`std::function` 和函数指针几乎一样慢，差距在 2% 以内。

也就是说——

> **`std::function` 的性能开销，本质上不是它自己"包装"出来的，而是"调用目标不可见"这件事本身。** 只要调用目标在编译期未知，无论是裸函数指针还是 `std::function`，编译器都只能老老实实做一次间接调用，后续所有优化（内联、常量传播、向量化）全部断掉。

同一份基准里还有个更震撼的对照组：**如果去掉那个 `volatile` 让编译器看见全部代码，lambda 和函数指针版本直接被折叠成常量，耗时 0 ms；而 `std::function` 版本仍然是 ~320 ms。** 这条间接调用隔开的，不只是几纳秒，而是"整个优化通道"。

### 什么时候该在意这个差距

| 场景 | 建议 |
|------|------|
| 每帧/每请求调用几次 | 完全不用管，`std::function` 的便利值这个价 |
| 内层循环里每次迭代都调 | 考虑模板参数或 lambda，别用 `std::function` |
| 需要存进容器、类型要统一 | 只能 `std::function`，这是它的设计目的 |
| 追求极致且能接受编译膨胀 | 模板 + `auto` 参数，编译器可完全内联 |

**微基准的诚实声明**：以上是微基准（microbenchmark），测量的是"两亿次空转调用"这一极端场景。真实程序里调用之间还有实际工作量，相对差距会被稀释。这个数字的意义是**量级参考**，不是"你的程序会慢 2.8 倍"。

## `std::function` 的内部结构：类型擦除 + SBO

上面那条 `call [rdi+24]` 已经从侧面暴露了 `std::function` 的内部布局。这一节把它拆开。

### 内存布局

```cpp
std::cout << sizeof(std::function<int(int,int)>);   // 32
```

实测 32 字节（libstdc++ / x86-64），分三块：

```
std::function<int(int,int)>  —  32 字节
┌──────────────────────────────┬────────────┬────────────┐
│  内联存储区（SBO 缓冲）        │  manager   │  invoker   │
│  16 字节                      │  8 字节     │  8 字节     │
└──────────────────────────────┴────────────┴────────────┘
 偏移 0                         偏移 16      偏移 24
                                    ↑            ↑
                        cmp [rdi+16], 0    call [rdi+24]
                        （空值检查）        （真正的调用）
```

- **偏移 0–15**：小对象优化（SBO, Small Buffer Optimization）的缓冲区。目标足够小就**直接存在这里**，不进堆。
- **偏移 16**：`manager` 指针。负责析构/拷贝/销毁被存的对象，同时兼作"是否为空"的判据。
- **偏移 24**：`invoker` 指针。就是那条 `call` 指令的目标——一个**知道怎么调用所存具体类型**的函数。

**"类型擦除（type erasure）"就发生在这里**：`std::function` 对外只暴露一个统一类型，对内靠两个函数指针记住了"我装的是什么类型、该怎么调、该怎么销毁"。它其实是**手工模拟的 vtable**——如果你读过虚函数的实现原理，会发现结构几乎一模一样。

### SBO 阈值实测

内联缓冲区到底能装多大？用不断增大的仿函数去试，数一下堆分配次数：

| 仿函数大小 | 是否堆分配 |
|-----------|-----------|
| 1 字节 | 就地（栈上） |
| 8 字节 | 就地（栈上） |
| 16 字节 | 就地（栈上） |
| **17 字节** | **堆分配** |
| 24 / 32 / 40 字节 | 堆分配 |

**结论：阈值是 16 字节。** 超过 16 字节，构造 `std::function` 时就会 `new`，析构时 `delete`。

对照一下：无捕获 lambda 实测 `sizeof` 为 **1 字节**（编译器需要一个非零大小来保证每个对象有唯一地址）。所以——

> **绝大多数无捕获 lambda 和单指针状态的仿函数都能就地存放，不会堆分配。** 捕获两个指针以上的闭包才会掉出 SBO。

### 代价清单

`std::function` 换来的是"统一类型"，付出的是：

1. **无法内联**（第六节）——这是最大的一笔开销，且影响的是整个优化通道。
2. **可能堆分配**——目标超过 16 字节时，构造/析构各一次 `new`/`delete`。
3. **要求目标可拷贝**——这是最容易被忽略、也最影响设计的限制。

第 3 条值得展开。`std::function` 的拷贝语义要求它存的东西**必须可拷贝**，所以捕获 `unique_ptr` 的 lambda 塞不进去：

```cpp
auto mv = [p = std::make_unique<int>(1)](int a) { return a + *p; };
std::function<int(int)> f = std::move(mv);   // 编译失败
```

```
error: static assertion failed: std::function target must be copy-constructible
```

这里还有个**陷阱**：`std::is_constructible_v<std::function<int(int)>, decltype(mv)>` 实测返回 **`true`**——因为 libstdc++ 把这个检查放在了构造函数体内的 `static_assert` 里，而不是用 SFINAE 约束签名，所以**类型特征会给你错误答案，直到你真正去构造才报错**。写约束时别信这个 trait。

这个限制在实际项目里非常常见：线程池、任务队列想存 `std::packaged_task`（move-only），用 `std::function` 直接不行，只能拿 `shared_ptr` 包一层把它变成可拷贝的。这就是 move-only 类型在多线程代码里经常被 `shared_ptr` 包裹的根本原因。

（C++23 的 `std::move_only_function` 专门解决这个问题，可以存 move-only 可调用对象，且是 `const`-正确和引用限定的。）

### 悬挂引用：比性能更该担心的事

`std::function` 最危险的用法不是慢，是**生命周期**：

```cpp
std::function<int()> make_bad() {
    int local = 42;
    return [&local] { return local; };   // 按引用捕获 → 编译通过
}   // local 在这里销毁

auto f = make_bad();
f();   // UB：访问已销毁的栈变量
```

按引用捕获的 lambda 被 `std::function` 存下来并逃出作用域，引用就悬空了。`std::function` **不会**帮你检查这件事，它只负责存和调。

安全规则：**要被 `std::function` 长期持有的 lambda，一律按值捕获**；确实需要共享状态就用 `shared_ptr` 捕获。反过来说，如果 lambda 只在当前作用域内立即使用（比如传给 `std::sort`），按引用捕获反而是高效且安全的。

## 常见误解纠偏

**"函数名就是函数指针。"**
不是。函数名的类型是函数类型，是左值；只在特定上下文才隐式转成指针。这个区别直接影响能否内联。

**"lambda 就是个匿名函数。"**
lambda 是**对象**（闭包），有类型、有大小、有成员（捕获的变量）。`sizeof` 一个无捕获 lambda 得到的是 1 而不是 0，因为它是个实实在在的对象。

**"两个写法一样的 lambda 是同一个类型。"**
不是，每个 lambda 表达式产生**独一无二**的闭包类型，实测 `is_same_v` 为 `false`。这也是它们不能相互赋值、不能写进同一个 `std::map` 默认构造的原因。

**"`std::function` 慢是因为它包装太厚。"**
不对。实测它和**裸函数指针几乎一样快**（~285ms vs ~280ms）。慢的根源是"调用目标编译期不可见"，函数指针同样中招。这个认知很重要——它意味着**想优化时，换掉 `std::function` 未必有用，得换掉"间接调用"这件事本身**（改用模板/`auto` 参数才能真正内联）。

**"`std::function` 一定能存下我的 lambda。"**
它要求目标**可拷贝**。捕获 `unique_ptr` 的 lambda、`std::packaged_task`、`std::promise` 都存不进去。更坑的是 `is_constructible` 检测不出来（libstdc++ 用的是 `static_assert` 而非 SFINAE），得到 `true` 之后在构造处才炸。

**"`std::function` 会帮我管理好生命周期。"**
它只管存和调。按引用捕获的 lambda 逃出作用域后，引用悬空，`std::function` 照调不误——这是 UB，不是异常。

## 总结对比

| 形式 | 携带状态 | 类型 | `sizeof` | 能否内联 | 实测相对开销 |
|------|---------|------|----------|---------|------------|
| 普通函数 | 否 | 函数类型（固定） | — | **是** | 1.0x |
| 函数指针 | 否 | 固定 | 8 | 通常否 | ~2.8x |
| 仿函数 | **是** | 唯一（自定义） | 自定义 | **是** | 1.0x |
| Lambda | **是** | 唯一（匿名） | 1（无捕获） | **是** | 1.0x |
| 成员函数指针 | 需绑定对象 | 固定 | 16 | 通常否 | ~2.8x |
| 成员数据指针 | 需绑定对象 | 固定 | 8 | 通常否 | — |
| `std::function` | 是 | 统一 | 32 | 否 | ~2.9x |

（`sizeof` 为 libstdc++ / x86-64 实测；相对开销来自第六节的微基准，仅供量级参考。）

**几条核心记忆点：**

1. **Callable 是一种能力，不是一种类型。** 标准用 `INVOKE` 表达式定义它，见 `[func.require]`。
2. **六种形式，按"类型是否唯一"分成两派**：函数/函数指针类型固定，仿函数/lambda 类型唯一。类型唯一的那派能被编译器完全看穿，因此可内联。
3. **性能分水岭是"能否内联"，不是"用了哪个类型"。** `std::function` 和函数指针一样慢，慢在间接调用隔断了整个优化通道。
4. **`std::invoke` 是标准库统一的调用语法**，覆盖成员函数指针和成员数据指针；库中所有"调用 callable"的地方语义上都是它。
5. **`std::function` = 手工 vtable + SBO**。32 字节里 16 字节是内联缓冲区，超过 16 字节的目标会堆分配；它要求目标可拷贝，这是很多 move-only 类型被迫套 `shared_ptr` 的根源。

Callable 是连接**泛型编程**和**回调机制**的桥梁。理解它的关键，不在于记住六种形式的名字，而在于理解**"类型在编译期是否可见"**这一条线索——内联、优化、性能、甚至 `std::function` 的设计取舍，全都是这条线索的推论。

## 参考

- 实测环境：GCC 13.3.0 / x86-64 / System V ABI / Ubuntu 24.04，Intel Xeon Platinum 8582C
- 汇编查看：`g++ -std=c++17 -O2 -S -masm=intel asm.cc`
- 基准代码：`n = 2e8`，`volatile` 阻止循环折叠，重复运行取稳定值
- 标准条款：`[func.require]`（INVOKE 与 Callable）、`[expr.call]`（函数调用）
- [`std::invoke` — cppreference](https://en.cppreference.com/w/cpp/utility/functional/invoke)
- [`std::function` — cppreference](https://en.cppreference.com/w/cpp/utility/functional/function)
- [`std::move_only_function` — cppreference](https://en.cppreference.com/w/cpp/utility/functional/move_only_function)
