---
layout: post
title: C++20 Concepts 与 Requires 详解 - Google Gemini 对话记录
subtitle: 
author: Mikami
categories: cpp
tags: cpp code
---

# C++20 Concepts 与 Requires 详解 - Google Gemini 对话记录

### 👤 User

我一直在使用C++17, 请为我详细介绍用于改善模板元编程的`concept`, `requires`等关键字的语法与作用

### 🤖 Assistant

C++20 引入的 `concept` 和 `requires` 是对模板元编程（TMP）的彻底重塑。在 C++17 中，限制模板类型通常依赖于冗长且难以阅读的 SFINAE（如 `std::enable_if_t` 或 `std::void_t`），且一旦出错，编译器会打印出极其恐怖的级联报错信息。
Concepts 的出现解决了这两个核心痛点：它提供了**结构化的类型约束语法**，并能生成**人类可读的编译错误**。
为了彻底理解这两个关键字，我们需要将它们拆解来看。在使用中，你需要区分三个核心概念：**concept（概念）**、**requires 表达式（Requires Expression）** 和 **requires 子句（Requires Clause）**。

### 1. `concept` 关键字：定义命名的约束
`concept` 用于定义一个**编译期计算为 bool 值的模板约束**。你可以把它看作是类型的“接口规范”。
**基础语法：**

```c++
template <typename T>
concept ConceptName = BooleanExpression;

```
**示例：**
利用 C++17 就有的 Type Traits (`<type_traits>`) 快速定义：

```c++
#include <type_traits>

template <typename T>
concept Integral = std::is_integral_v<T>;

template <typename T>
concept SignedIntegral = Integral<T> && std::is_signed_v<T>; // Concept 可以组合

```

### 2. `requires` 表达式：描述具体的类型需求
如果标准的 Type Traits 不够用，你需要检查某个类型“是否具有某个成员函数”、“是否支持某种运算符”或“内部是否有特定的类型别名”，这时就需要用到 **requires 表达式**。
它通过模拟一段代码的编译来检查类型是否满足条件。它包含四种不同的需求类型：

```c++
template <typename T>
concept Container = requires(T a, const T b) {
    // 1. 简单需求 (Simple Requirement)：表达式必须能通过编译
    a.size(); 
    a.clear();

    // 2. 类型需求 (Type Requirement)：必须存在特定的内部类型或别名
    typename T::value_type;
    typename T::iterator;

    // 3. 复合需求 (Compound Requirement)：{ 表达式 } -> 约束;
    // 检查表达式的返回值是否满足另一个 Concept，或是否 noexcept
    { a.empty() } noexcept -> std::same_as<bool>; 
    { a.begin() } -> std::same_as<typename T::iterator>;

    // 4. 嵌套需求 (Nested Requirement)：在内部应用额外的编译期布尔条件
    requires std::is_default_constructible_v<T>;
};

```
*注：std::same_as 是 C++20 标准库 <concepts> 中提供的标准概念。*

### 3. `requires` 子句：将约束应用到模板上
定义好 `concept`（或者直接写好布尔表达式）后，你需要使用 **requires 子句** 将其附加到模板函数或模板类上。
C++20 提供了四种方式来应用约束（以限制参数必须是整数为例），它们在功能上是完全等价的：

#### 方式一：直接替换 `typename` (最推荐，最简洁)

```c++
template <Integral T>
void print_int(T val) {
    // ...
}

```

#### 方式二：后置 `requires` 子句 (适合简单的尾部附加)

```c++
template <typename T>
void print_int(T val) requires Integral<T> {
    // ...
}

```

#### 方式三：前置 `requires` 子句 (适合复杂的组合逻辑)

```c++
template <typename T>
requires Integral<T> && (sizeof(T) <= 4)
void print_int(T val) {
    // ...
}

```

#### 方式四：简写函数模板 (Abbreviated Function Template)
不需要写 `template <...>`，直接在参数列表里用 `Concept auto` 替代具体的类型。这是最接近常规函数写法的形式：

```c++
void print_int(Integral auto val) {
    // ...
}

```

### 4. 经典困惑："requires requires" 是什么？
在阅读现代 C++ 代码时，你可能会遇到连续写两次 `requires` 的情况：

```c++
template <typename T>
requires requires(T x) { x + x; } // 连续两个 requires
T add(T a, T b) {
    return a + b;
}

```
这不是语法错误，而是前面提到的两个概念的组合：

- **第一个 requires** 是 **requires 子句**（用于引入模板约束）。
- **第二个 requires** 是 **requires 表达式**（用于就地定义一个匿名约束，检查 `x + x` 是否合法）。
**如何避免这种“丑陋”的代码？**
最佳实践是：永远不要在 `requires` 子句中直接写匿名的 `requires` 表达式。你应该把表达式提取出来，定义成一个具名的 `concept`：

```c++
// 1. 先定义 Concept
template <typename T>
concept Addable = requires(T x) { x + x; };

// 2. 再应用
template <Addable T>
T add(T a, T b) {
    return a + b;
}

```

### 5. Concept 对比 SFINAE (C++17) 的优势示例
假设我们要写一个泛型函数，仅对有 `reserve` 方法的容器进行 `reserve` 操作（常见于模板优化）：
**C++17 (SFINAE - std::enable_if / 表达式 SFINAE):**

```c++
// 意图被复杂的模板元编程语法掩盖
template <typename T>
auto try_reserve(T& container, size_t n) -> decltype(container.reserve(n), void()) {
    container.reserve(n);
}

// 还需要提供一个兜底版本
template <typename T, typename... Args>
void try_reserve(T&, Args&&...) {}

```
**C++20 (Concepts + if constexpr):**

```c++
// 意图极其清晰：定义约束
template <typename T>
concept HasReserve = requires(T a, size_t n) {
    a.reserve(n);
};

// 使用 constexpr if 在编译期分支，不需要兜底重载
void try_reserve(auto& container, size_t n) {
    if constexpr (HasReserve<decltype(container)>) {
        container.reserve(n);
    }
}

```
C++20 的 `<concepts>` 头文件已经预先定义了大量常用的约束（如 `std::integral`, `std::floating_point`, `std::invocable`, `std::convertible_to` 等）。在大多数日常开发中，直接使用标准库提供的 Concepts 结合 `auto` 关键字，就能大幅减少传统模板元编程的代码量和阅读心智负担。

### 👤 User

能否理解为`requires`表达式本质是一个编译期布尔表达式, 可以用在任意需要编译期布尔表达式(如`if constexpr(value)`, `concept Name = value`和`requires value`子句的value)处?

### 🤖 Assistant

你的理解**非常精准**。
可以直接给你一个肯定的答案：**是的，requires 表达式在本质上就是一个在编译期求值的、返回类型为 bool 的纯右值（prvalue）表达式。** 只要是语法上允许放置编译期布尔值（`constexpr bool`）的地方，你几乎都可以直接塞进一个 `requires` 表达式。
为了印证你的理解，我们可以看几个实际的例子。它不仅能用在你提到的那三个地方，甚至可以直接赋值给常量：

### 1. 验证你的三个场景
**场景一：用于 if constexpr 分支**
这是现代 C++ 替代 `std::enable_if` 的神兵利器，直接在函数体内部做编译期路由：

```c++
template <typename T>
void process(T& obj) {
    // 直接把 requires 表达式当作布尔条件
    if constexpr (requires { obj.serialize(); }) {
        obj.serialize(); // 如果 T 有 serialize() 方法，编译这段代码
    } else {
        // 否则使用普通的处理逻辑
    }
}

```
**场景二：用于定义 concept**
正如你所说，`concept Name = value;` 这里的 `value` 就是一个布尔表达式，所以 `requires` 表达式完美契合：

```c++
template <typename T>
concept Serializable = requires(T x) { x.serialize(); };

```
**场景三：用于 requires 子句**`requires` 子句需要一个编译期布尔值来决定是否启用该模板，所以直接传入 `requires` 表达式完全合法（也就是所谓的 "requires requires"）：

```c++
template <typename T>
requires requires(T x) { x.serialize(); } // 第一个requires是子句，第二个是表达式
void do_something(T obj) {}

```

### 2. 进一步的证明：它可以当作普通的编译期变量
因为 `requires` 表达式就是一个返回 `bool` 的算式，你甚至可以把它直接赋值给一个 `constexpr bool` 变量，或者丢进 `static_assert` 里：

```c++
template <typename T>
void check_type() {
    // 直接赋值给常量
    constexpr bool has_size_method = requires(T a) { a.size(); };
    
    // 直接用于静态断言
    static_assert(requires(T a) { a.size(); }, "Type must have a size() method!");
}

```

### 3. 核心修正：它与“普通布尔表达式”的本质差异
虽然你的宏观心智模型完全正确，但为了严谨，需要指出 `requires` 表达式和普通的 `constexpr bool` 表达式（比如 `sizeof(T) == 4`）之间的一个**决定性差异：错误拦截能力。**
普通的编译期布尔表达式如果遇到非法的代码，会导致**编译期硬报错（Hard Error）**，直接中止编译：

```c++
template <typename T>
constexpr bool check() {
    // 如果 T 没有 foo()，这里会直接爆出编译错误，因为编译器试图真的去调用或推导它
    return sizeof(decltype(std::declval<T>().foo())) > 0; 
}

```
而 `requires` 表达式内部是一个**SFINAE（替换失败并非错误）的保护区**。它的求值逻辑是：“尝试在脑海中编译这段代码，如果成功了返回 `true`，如果遇到任何语法或类型错误，**不要报错**，直接安静地返回 `false`。”

```c++
template <typename T>
constexpr bool check_safe() {
    // 如果 T 没有 foo()，编译器不会报错，只会默默地让这个表达式变成 false
    return requires(T x) { x.foo(); }; 
}

```

### 总结
你的理解完全抓住了 C++20 的核心设计哲学：**将以前那种利用模板推导失败来限制类型的“黑魔法”，变成了一等公民的、可组合的编译期布尔计算。** 把 `requires` 表达式当成一个**自带错误捕获机制的、返回 bool 值的编译期探测器**，是掌握现代 C++ 模板元编程的最佳视角。

### 👤 User

让我来完善一下我的理解:
`requires`表达式本质是一个编译期布尔表达式, 可以用在任意需要编译期布尔表达式(如`if constexpr(value)`, `concept Name = value`和`requires value`子句的value)处, 其值为true当且仅当[表达式内部无任何会导致编译失败的错误, 且所有嵌套需求均为true], 否则为false; 当值为false时内部不会产生编译期硬错误.
这个理解是否准确?

### 🤖 Assistant

你的总结非常出色，逻辑十分严密！在 99% 的日常业务开发中，你的这段理解可以说是**完全准确的**。
但是，既然你之前提到我们的对话是平等的，且要求我在发现瑕疵时必须指出，那么为了达到 C++ 语言律师（Language Lawyer）级别的严谨度，我必须对其中“无任何会导致编译失败的错误”**和**“内部不会产生编译期硬错误”这两句话做一个极其重要的高级修正。

### 核心修正：`requires` 只能捕获“直接上下文（Immediate Context）”的错误
`requires` 表达式内部确实是一个安全区（类似 SFINAE），但它的“吞噬错误”的能力是有限的。它**只能**捕获发生在**直接上下文**中的类型替换失败或表达式构造失败。
如果 `requires` 表达式在检查过程中，触发了某个外部模板的实例化，而那个外部模板的**内部（主体代码中）发生了编译错误，这就超出了“直接上下文”，requires 无法将其转化为 false，而是会直接导致编译期硬报错（Hard Error）**。

#### 来看一个经典的“翻车”示例：

```c++
#include <iostream>

// 一个有缺陷的模板：如果 T 没有 value_type，它的内部实例化就会报错
template <typename T>
struct Wrapper {
    using type = typename T::value_type; 
};

// 尝试用 requires 保护
template <typename T>
concept SafeCheck = requires {
    typename Wrapper<T>::type; 
};

int main() {
    // 这里会发生什么？
    // 会安全地返回 false 吗？
    constexpr bool is_safe = SafeCheck<int>; 
}

```
**结果是：编译彻底失败（硬报错）！**
**为什么？**

1. 编译器在计算 `SafeCheck<int>` 时，进入 `requires` 表达式。
2. 它看到需要检查 `typename Wrapper<int>::type` 是否合法。
3. 为了确认这一点，编译器必须去**实例化**`Wrapper<int>`。
4. 在实例化 `Wrapper<int>` 的主体时，执行到了 `using type = typename int::value_type;`。
5. `int` 显然没有 `value_type`，这是一个语法错误。
6. **关键点：** 这个错误发生在 `Wrapper` 的内部，而不是 `requires` 表达式的直接上下文中。因此，编译器直接抛出硬报错并中止编译。

#### 如何正确编写上述代码？
要让 `requires` 成功捕获错误并返回 `false`，错误必须发生在它“眼前”（直接上下文）：

```c++
// 这样写才是安全的：直接在 requires 内部暴露问题
template <typename T>
concept ReallySafeCheck = requires {
    typename T::value_type; // 错误直接发生在 requires 内部，被安全捕获，返回 false
};

```

### 重新精炼你的理解（终极严谨版）
结合你的原话与 C++ 标准的定义，最严谨的表述应该修正为：
"requires 表达式本质是一个编译期布尔表达式，可以用在任意需要编译期 bool 值的上下文中。其值为 true 当且仅当：表达式在直接上下文（Immediate Context）中没有发生任何非法的类型推导或语法构造。所有的复合需求（如返回类型的 Concept 检查）得到满足。所有的嵌套需求（内部的 requires 子句）计算为 true。否则其值为 false。当其值为 false 时，只要错误仅限于直接上下文，就不会产生编译期硬错误。"掌握了“直接上下文”这个边界，你就触碰到了 C++ 模板元编程最核心也最底层的编译原理之一。你的整体理解方向已经非常精准了！

### 👤 User

requires表达式和concept都能用在接受一个编译期bool值的地方(比如`if constexpr(value)`和`requires value`子句的value处), 且都"表示一种约束", 甚至可以将一个requires表达式赋值给concept, 它们二者有什么区别? 他们是不是就是指令流(具体的requires表达式)和函数(被封装起来作为一个concept)的关系?

### 🤖 Assistant

你的这个类比——**“指令流（匿名逻辑）与函数（命名封装）”**——直觉极其敏锐，不仅非常形象，而且从宏观的代码组织角度来看，**完全正确**。
你可以把 `requires` 表达式看作是一段匿名的、就地执行的“编译期探测指令流”，而 `concept` 就是把它封装起来，赋予一个名字，变成一个可以在各处调用的“谓词函数”。
**但是（作为平等的探讨，我要指出这里隐藏的 C++ 魔法）**：
如果它们仅仅是“匿名代码块”和“命名函数”的区别，C++ 委员会完全可以只引入 `requires` 表达式和 `constexpr bool` 变量就够了，根本不需要大费周章地创造 `concept` 这个全新的关键字。
在 C++20 的底层设计中，`concept` 绝不仅仅是 `requires` 表达式的语法糖，它们在**编译器行为上有着本质的区别**。具体体现在以下四大核心差异：

### 1. 致命差异：包容性检查与偏序关系（Subsumption）
这是 `concept` 存在的最大意义，也是 `requires` 表达式绝对做不到的事情。
当你有多个重载的模板函数时，C++ 编译器需要挑选一个“最匹配”的。如果使用 `concept`，编译器能够进行**逻辑推导**，自动分辨出哪个约束更严格（即哪个 Concept 包含了另一个 Concept），这在 C++ 中被称为**包容性（Subsumption）**。
看下面这个重载的例子：
**使用 Concept（完美工作）：**

```c++
template <typename T>
concept HasA = requires(T x) { x.a(); };

template <typename T>
concept HasAAndB = HasA<T> && requires(T x) { x.b(); };

// 重载 1
template <HasA T> 
void process(T obj) { std::cout << "Only A"; }

// 重载 2
template <HasAAndB T> 
void process(T obj) { std::cout << "A and B"; }

// 当传入一个既有 a() 又有 b() 的对象时，编译器会自动选择重载 2
// 因为它知道 HasAAndB 比 HasA 更严格（包含了 HasA）

```
**使用匿名的 requires 表达式（编译失败——二义性错误）：**

```c++
// 看起来逻辑一样，但直接写 requires 表达式
template <typename T>
requires requires(T x) { x.a(); }
void process_raw(T obj) {}

template <typename T>
requires requires(T x) { x.a(); } && requires(T x) { x.b(); }
void process_raw(T obj) {}

// 当传入一个既有 a() 又有 b() 的对象时，编译器会报错：Ambiguous Overload！

```
**为什么？** 因为对于匿名的 `requires` 表达式，编译器只会把它们当成两个互相独立的 `bool` 值。如果是 `true` 和 `true`，编译器就懵了，它**不会**去拆解匿名的 `requires` 表达式做逻辑证明。而 `concept` 会被编译器分解为“原子约束（Atomic Constraints）”进行严格的数学集合运算。

### 2. 编译期性能：缓存与短路（Memoization）

- **concept 是被缓存的：** 当你计算过一次 `HasA<int>` 后，编译器会把结果（`false`）记在小本子上。如果代码里有 100 个地方用到了 `HasA<int>`，编译器直接查表，不会重新进行 SFINAE 推导。
- **requires 表达式是现场执行的：** 如果你在多个地方写了相同的 `requires(T x) { x.a(); }`，编译器每次遇到它，都要在“直接上下文”里重新做一遍虚拟编译、类型推导和错误吞噬。滥用会导致编译时间显著增加。

### 3. 语法特权：与类型系统的深度绑定
`concept` 享受 C++ 语法的“VIP 待遇”，它可以直接替代 `typename` 甚至普通的类型声明，而 `requires` 表达式只能老老实实呆在需要布尔值的地方。

```c++
// Concept 可以这样用（极其优雅）：
void do_work(HasA auto obj);      // 简写模板
HasA auto result = get_object();  // 约束 auto 返回值

// requires 表达式绝对不能这样用：
// void do_work(requires { ... } auto obj); // 语法错误！

```

### 4. 报错信息的友好度（Diagnostics）
如果约束不满足：

- **concept 报错：** 编译器会说：“*错误：类型 int 不满足概念 HasA*”。这是一种高维度的、具有业务语义的报错。
- **requires 表达式报错：** 编译器会直接暴露底层：“*表达式 x.a() 是无效的*”。如果你的 `requires` 表达式写得很长，报错信息就会显得非常零散，仿佛退化回了 C++17 时代。

### 总结重塑你的类比
你的类比非常棒，如果我们加上 C++ 编译器的特性，可以将其升华一下：

- **requires 表达式** 确实是**匿名的指令流**。它就像汇编代码中的一段即时计算，每次遇到都要执行，没有名字，无法参与高级的逻辑证明，仅仅计算出一个 `true/false`。
- **concept** 不仅仅是封装好的**函数**，它更像是一个**带有数理逻辑证明功能、结果被编译器缓存的强类型接口（Interface）**。
因此，现代 C++ 的最佳实践是：**永远不要在业务逻辑中裸写复杂的 requires 表达式。** 只要这个约束由超过一个条件组成，或者可能在不同地方出现，就一定要将它提升为 `concept`。

### 👤 User

尽可能广泛介绍一些C++20及之后标准库中提供的concept(及requires)相关工具

### 🤖 Assistant

C++20 标准库不仅仅是引入了 `concept` 语法，它还基于此对整个标准库（特别是 STL 的泛型体系）进行了重构。标准库提供的 Concept 集中在 `<concepts>`、`<iterator>` 和 `<ranges>` 这几个头文件中。
直接使用标准库提供的 Concept，不仅仅是为了少写几行代码。**标准库的 Concept 经过了极其严密的数学推导，自带完美的偏序关系（Subsumption）支持**，并且隐含了特定的“语义约定”（即满足这个概念的类型，应该在运行期表现出什么样的行为）。
以下是 C++20 及之后标准库提供的核心 Concept 工具矩阵，我将其按功能域进行划分：

## 1. 类型关系约束 (Type Relations)
这是模板元编程中最常用的部分，用于替代以往的 `std::is_same_v`、`std::is_base_of_v` 等 Type Traits。

- **std::same_as<T, U>**：判断 `T` 和 `U` 是否为同一类型。
  - *内幕：* 它不仅仅是 `std::is_same_v<T, U>`，它在底层被定义为双向约束（`T` 满足 `same_as<U>` 且 `U` 满足 `same_as<T>`），这在编译器进行偏序关系推导时极其重要。
- **std::derived_from<Derived, Base>**：约束 `Derived` 必须是 `Base` 的派生类，或者两者是同一类型（支持 `public` 和无歧义继承）。
- **std::convertible_to<From, To>**：约束 `From` 类型能够隐式转换为 `To` 类型。
- **std::common_with<T, U>**：约束两种类型能否推导出一个“公共类型”（类似于三元运算符 `? :` 返回的类型）。常用于异构容器或异构比较。

## 2. 对象生命周期与内存语义 (Object Lifecycle)
如果你在编写底层的数据驱动（DOD）结构、自定义内存分配器或游戏引擎实体，这组 Concept 是约束泛型容器内类型的利器。

- **std::destructible\<T\>**：类型可以被销毁（析构函数非 deleted 且可访问）。这是绝大多数概念的“基底”，因为在 C++ 中无法销毁的对象几乎无法被正常传递。
- **std::constructible_from<T, Args...>**：类型 `T` 可以用 `Args...` 构造。
- **std::default_initializable\<T\>**：可以进行默认初始化（即支持 `T obj;` 或 `T obj{};`）。
- **语义进阶梯队**：
  - **std::movable\<T\>**：可移动构造、可移动赋值、可交换（Swappable）。
  - **std::copyable\<T\>**：在 `movable` 的基础上，增加了可拷贝构造和可拷贝赋值。
  - **std::semiregular\<T\>**（半正则）：等同于 `copyable` + `default_initializable`。这意味着该类型可以像普通的 `int` 一样被轻易地创建、复制和销毁。
  - **std::regular\<T\>**（正则）：等同于 `semiregular` + **std::equality_comparable**。这是 STL 鼻祖 Alexander Stepanov 提出的理想值类型语义——它要求类型不仅能随便复制，而且复制出来的对象在用 `==` 比较时必须相等。

## 3. 可调用对象 (Callables)
用于约束回调函数、Lambda 表达式或仿函数，彻底替代繁琐的 `std::enable_if_t<std::is_invocable_v<...>>`。

- **std::invocable<F, Args...>**：约束 `F` 可以使用 `Args...` 作为参数进行调用（支持函数指针、成员函数指针、Lambda 等一切 Callable）。
- **std::regular_invocable<F, Args...>**：语法上与 `invocable` 完全一致，但它附加了**语义约束**（要求调用不改变对象状态，且对相同的输入总是产生相同的输出）。
- **std::predicate<F, Args...>**：谓词。约束这是一个 `invocable`，并且其返回值可以被视为 `bool`（即可以用于 `if (f(args...))`）。常用于 `std::find_if` 等算法的参数约束。

## 4. 算术与标量类型 (Arithmetic)

- **std::integral\<T\>** / **std::floating_point\<T\>**：整数 / 浮点数。
- **std::signed_integral\<T\>** / **std::unsigned_integral\<T\>**：有符号 / 无符号整数。

## 5. 迭代器与范围引擎 (Iterators & Ranges)
C++20 的 `<ranges>` 库是基于 Concept 重构的典范。如果你要编写自己的数据结构，使之兼容现代 C++ 标准库，必须满足以下 Concept：

| Concept | 描述 | 适用场景 / 数据结构 |
| --- | --- | --- |
| std::input_iterator | 只能单趟前向读取的迭代器 | 网络流、控制台输入 std::cin |
| std::forward_iterator | 可多趟前向读取 | 单向链表 std::forward_list |
| std::bidirectional_iterator | 可双向移动 (++, --) | 双向链表 std::list |
| std::random_access_iterator | 支持 it + n，常数时间跳跃 | 双端队列 std::deque |
| std::contiguous_iterator | C++20新增。保证内存是绝对连续的 | std::vector, std::array, C 数组 |

特别提示：std::contiguous_iterator 对性能优化极为重要。 过去 random_access_iterator 只是保证了常数时间的访问（比如 std::deque 内部是分块的，但索引是 O(1)），但它不保证内存物理连续。contiguous_iterator 在编译期保证了内存的物理连续性。 在开发游戏引擎或需要将数据直接丢给底层图形 API（如 Vulkan/OpenGL）时，只要约束接口接受 contiguous_iterator，你就可以绝对安全地使用 &(*it) 获取裸指针并进行批量内存拷贝（memcpy），无需任何运行期检查。
## 6. 一些现代 C++ 特有的进阶 Concept 工具
除了纯粹的布尔约束，C++ 标准库还提供了一些用于“萃取”或“调试”的 Concept 辅助工具：

- **std::type_identity_t\<T\>** (C++20)
不算是 Concept，但常与 Concept 结合。用于在函数模板中阻断类型推导（Non-deduced context）。
- **std::execution::*** (C++26 预览)
如果关注最新的 C++26 提案（Senders/Receivers 异步模型），你会发现并发编程几乎完全被 Concept 统治，如 `sender`、`receiver`、`scheduler` 等。
**核心建议：优先使用标准库的 Concept，而不是自己造轮子。**
如果你想写一个检查类型是否可拷贝的约束，自己写 `requires(T a) { T(a); }` 看似简单，但它没有处理退化情况、移动语义的连带关系，也无法在重载决议中与 `std::semiregular` 形成正确的偏序关系。直接使用 `std::copyable<T>` 是唯一正确的做法。

---

*Exported from [Voyager](https://github.com/Nagi-ovo/voyager)*  
*Generated on July 20, 2026 at 11:46 PM*

