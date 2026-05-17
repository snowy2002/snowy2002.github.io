---
title: C++学习笔记（一）
filename: 20260517-cpp-study-notes-001
tags:
  - cpp
  - 学习笔记
categories:
  - C++
date: 2026-05-17
description: 深入理解 C++ 移动语义、左值右值、完美转发与特殊成员函数，配合代码示例和面试题。
articleGPT: 本文从内存分区出发，系统讲解 C++ 中左值/右值/纯右值/将亡值的区别，移动语义的动机与实现，移动构造函数与移动赋值操作符的写法，RVO 返回值优化，以及 std::forward 完美转发的原理与应用。
share: true
delete: false
---

## 内存分区

在理解移动语义之前，先回顾 C++ 的内存布局。堆区（Heap）上的动态分配是移动语义要优化的核心场景——避免不必要的堆内存拷贝。

![C++ 内存分区详解](/images/posts/2026-05/内存分区.png)

- **栈区（Stack）**：局部变量、函数参数，自动分配释放，速度快
- **堆区（Heap）**：`new / malloc` 动态分配，需要手动管理，容易产生不必要的拷贝
- **全局/静态区**：全局变量、`static` 变量
- **常量区（.rodata）**：字符串字面量、`const` 全局变量
- **代码区（.text）**：存放编译后的机器指令

当一个类持有堆上的资源（如 `char* data = new char[100]`），拷贝意味着重新 `malloc + memcpy`，而移动只需要**转移指针**。这就是移动语义存在的意义。

## 左值与右值

C++ 中每个表达式都有一个**值类别（value category）**，最基本的分类是左值和右值。

### 左值（lvalue）

有名字、有地址、生命周期持续存在的对象。

```cpp
int x = 42;           // x 是左值
String s("Hello");    // s 是左值
int* p = &x;          // 能取地址 → 左值
```

### 右值（rvalue）

临时产生的值，没有持久的身份，即将被销毁。

```cpp
42                     // 字面量，右值
x + y                  // 算术表达式的结果，右值
String("Hello")        // 临时对象，右值
```

### 左值引用与右值引用

```cpp
int& ref = x;              // 左值引用：绑定到左值
int&& rref = 42;           // 右值引用：绑定到右值
String&& sref = String("Hi"); // 右值引用绑定临时对象
```

## 值类别详解：lvalue、prvalue、xvalue

C++ 标准将表达式的值类别进一步细分，用两个属性来区分：

- **有身份（identity）**：能取地址、能被多次引用
- **可移动（movable）**：资源可以被偷走

|  | 可移动 | 不可移动 |
|--|--------|----------|
| **有身份** | xvalue（将亡值） | lvalue（左值） |
| **无身份** | prvalue（纯右值） | （不存在） |

关系图：

```
           表达式
          /      \
      glvalue    rvalue
     (有身份)   (可移动)
      /    \     /    \
  lvalue  xvalue  prvalue
```

### 纯右值（prvalue）

无身份，可移动。临时产生的值，用完即弃。

```cpp
42                     // 字面量
x + y                  // 算术结果
String("Hello")        // 临时对象
true                   // 布尔字面量

// 不能取地址
// &42;             // 错误
// &String("Hello"); // 错误
```

C++17 起，prvalue 享受**强制拷贝消除**，不会产生临时对象，直接在目标位置构造。

### 将亡值（xvalue）

有身份，但被显式标记为可移动。

```cpp
std::move(s)                   // s 有名字，但声明放弃
static_cast<String&&>(s)       // 同上
std::move(s).data              // 访问右值的成员
```

### 最容易混淆的点：右值引用变量本身是左值

```cpp
void foo(String&& s) {
    // s 的类型是 String&&（右值引用）
    // 但 s 这个表达式是左值！因为它有名字

    String a = s;             // 拷贝构造！s 是左值
    String b = std::move(s);  // 移动构造，需要 move 再转一次
}
```

**有名字的东西都是左值**，不管它的声明类型是什么。

### 速记

```
左值：有名字能取地址        → 拷贝
纯右值：临时的，没名字      → 编译器直接优化掉（RVO）
将亡值：有名字但你说不要了  → 移动，偷资源
```

## 移动语义

### 解决的核心问题：不必要的拷贝

C++11 之前，把对象从 A 传到 B 只有**拷贝**一条路。即使原对象马上销毁，也得完整复制一份再析构。

```cpp
class String {
    char* data;
    size_t length;
public:
    String(const char* str) {
        length = strlen(str);
        data = new char[length + 1];
        strcpy(data, str);
    }
    ~String() { delete[] data; }
};
```

没有移动语义的时代：

```
String createString() {
    String s("Hello World");
    return s;
}
String greeting = createString();

// 流程：
// 1. 构造 s           → malloc + memcpy
// 2. 拷贝构造返回值    → malloc + memcpy  ← 浪费
// 3. 析构 s           → free
// 4. 拷贝构造 greeting → malloc + memcpy  ← 浪费
// 5. 析构返回值        → free
```

`"Hello World"` 被 malloc 了 **3 次**，其中 2 次完全是浪费。

### 有了移动语义之后

编译器发现源对象即将销毁（右值），调用移动构造函数，**偷走资源而非复制**：

```cpp
String(String&& other) noexcept {
    data = other.data;        // 偷指针，O(1)
    length = other.length;
    other.data = nullptr;     // 源对象置空
    other.length = 0;
}
```

从 3 次 malloc 变成 1 次。移动构造只是指针赋值，零成本。

### std::move

`std::move` **不移动任何东西**，它只是一个强制类型转换，把左值转成右值引用：

```cpp
String a("Hello");
String b = a;              // 拷贝，a 之后还能用
String c = std::move(a);   // 移动，a 被掏空
// 此后 a.data == nullptr，不应该再使用 a
```

## 特殊成员函数：移动构造与移动赋值

C++ 有 6 个特殊成员函数，移动构造和移动赋值是 C++11 新增的：

```
默认构造函数          T()
析构函数              ~T()
拷贝构造函数          T(const T&)
拷贝赋值操作符        T& operator=(const T&)
移动构造函数          T(T&&)              ← C++11
移动赋值操作符        T& operator=(T&&)   ← C++11
```

核心区别：**构造是创建新对象，赋值是覆盖已有对象**。赋值比构造多一步——要先释放自己原来的资源。

### 完整实现

```cpp
class String {
    char* data;
    size_t length;

public:
    // 普通构造
    String(const char* str = "") {
        length = strlen(str);
        data = new char[length + 1];
        strcpy(data, str);
    }

    // 析构
    ~String() { delete[] data; }

    // ========== 拷贝语义 ==========

    // 拷贝构造：深拷贝，分配新内存
    String(const String& other) {
        length = other.length;
        data = new char[length + 1];
        strcpy(data, other.data);
    }

    // 拷贝赋值：释放旧资源，再深拷贝
    String& operator=(const String& other) {
        if (this != &other) {
            delete[] data;
            length = other.length;
            data = new char[length + 1];
            strcpy(data, other.data);
        }
        return *this;
    }

    // ========== 移动语义 ==========

    // 移动构造：偷资源，源对象置空
    String(String&& other) noexcept {
        data = other.data;
        length = other.length;
        other.data = nullptr;
        other.length = 0;
    }

    // 移动赋值：释放旧资源 → 偷新资源 → 源对象置空
    String& operator=(String&& other) noexcept {
        if (this != &other) {
            delete[] data;          // 释放自己的旧资源
            data = other.data;      // 偷走资源
            length = other.length;
            other.data = nullptr;   // 源对象置空
            other.length = 0;
        }
        return *this;
    }
};
```

### 触发时机

```cpp
String a("Hello");
String b("World");

String c = a;               // 拷贝构造（用已有对象初始化新对象）
String d = std::move(a);    // 移动构造（用右值初始化新对象）

b = c;                      // 拷贝赋值（对象已存在，用另一个覆盖）
b = std::move(c);           // 移动赋值（对象已存在，用右值覆盖）
```

### noexcept 的重要性

移动构造和移动赋值应该标记 `noexcept`。原因：`std::vector` 扩容时，只有移动构造是 `noexcept` 的，才会用移动而非拷贝。

```cpp
class A { A(A&&) { } };                  // 无 noexcept → vector 扩容用拷贝
class B { B(B&&) noexcept { } };         // 有 noexcept → vector 扩容用移动
```

如果移动到一半抛异常，已搬走的元素无法恢复，容器的强异常安全保证会被破坏。所以标准库的策略是：**只有 noexcept 的移动才敢用**。

## RVO：返回值优化

RVO（Return Value Optimization）是编译器的优化：函数返回对象时，跳过拷贝/移动，直接在调用方的内存上构造。

```cpp
String createString() {
    return String("Hello");  // prvalue，C++17 起强制 RVO
}
String s = createString();   // 只有 1 次普通构造，零拷贝零移动
```

### RVO 的两种形式

| 类型 | 说明 | 示例 |
|------|------|------|
| **RVO** | 返回匿名临时对象 | `return String("Hello");` |
| **NRVO** | 返回有名字的局部变量 | `String s("Hello"); return s;` |

```cpp
// RVO — C++17 起强制生效
String f() { return String("Hello"); }

// NRVO — 不强制，但几乎所有编译器都会做
String g() {
    String s("Hello");
    return s;
}
```

### 重要：不要对 return 的局部变量用 std::move

```cpp
// 错误！阻止 NRVO，反而多一次移动
String bad()  { String s("Hi"); return std::move(s); }

// 正确：让编译器自动优化
String good() { String s("Hi"); return s; }
```

### 返回参数时必须用 std::move

局部变量 return 时会自动当右值处理，但**函数参数不享受这个待遇**：

```cpp
String g(String&& s) { return s; }              // 拷贝！s 是左值
String g(String&& s) { return std::move(s); }   // 移动，正确写法
```

## std::forward：完美转发

### 解决的问题

写转发函数时，参数的左值/右值属性会丢失：

```cpp
template<typename T>
void wrapper(T&& arg) {
    process(arg);   // arg 有名字 → 永远是左值 → 永远走拷贝！
}

wrapper(std::move(s));  // 传入右值，但 process 收到的是左值
```

`std::forward` 的作用：**原封不动地转发，左值进左值出，右值进右值出。**

```cpp
template<typename T>
void wrapper(T&& arg) {
    process(std::forward<T>(arg));  // 完美转发
}

wrapper(a);              // 左值 → forward 保持左值 → process(T&)
wrapper(std::move(a));   // 右值 → forward 保持右值 → process(T&&)
```

### 原理：万能引用 + 引用折叠

`T&&` 在模板中是**万能引用（universal reference）**，推导规则：

```cpp
template<typename T>
void wrapper(T&& arg);

String s("Hello");
wrapper(s);              // s 是左值 → T = String&  → T&& = String&
wrapper(std::move(s));   // 右值    → T = String   → T&& = String&&
```

引用折叠规则：

```
T& &   → T&     左 + 左 = 左
T& &&  → T&     左 + 右 = 左
T&& &  → T&     右 + 左 = 左
T&& && → T&&    右 + 右 = 右  ← 唯一产生右值引用的情况
```

### std::move vs std::forward

```
std::move(x)        — 无条件转成右值，"我不要了"
std::forward<T>(x)  — 有条件转发，"原来是什么就是什么"
```

### 典型用途：工厂函数和 emplace

```cpp
template<typename T, typename... Args>
unique_ptr<T> make_unique(Args&&... args) {
    return unique_ptr<T>(new T(std::forward<Args>(args)...));
}

make_unique<String>("Hello");       // 转发右值 → 普通构造
make_unique<String>(s);             // 转发左值 → 拷贝构造
make_unique<String>(std::move(s));  // 转发右值 → 移动构造
```

`vector::emplace_back` 也是相同原理：

```cpp
vector<String> v;
v.emplace_back("Hello");          // 转发 const char* → 直接构造
v.emplace_back(std::move(s));     // 转发右值 → 移动构造
```

## 函数限定符速查

函数参数列表 `)` 之后可以放多种限定符：

| 限定符 | 作用 | 示例 |
|--------|------|------|
| `const` | 不修改对象状态 | `int get() const;` |
| `noexcept` | 保证不抛异常 | `T(T&&) noexcept;` |
| `override` | 显式声明重写虚函数 | `void draw() override;` |
| `final` | 禁止继续重写/继承 | `void draw() final;` |
| `&` / `&&` | 限制调用者是左值/右值 | `T get() &&;` |
| `= default` | 使用编译器默认实现 | `T(T&&) = default;` |
| `= delete` | 禁用该函数 | `T(const T&) = delete;` |