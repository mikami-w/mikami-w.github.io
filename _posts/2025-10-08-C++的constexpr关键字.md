---
layout: post
title: C++的constexpr关键字
subtitle: 强大的编译期优化者
author: Mikami
categories: cpp
tags: cpp code
---

### 1. C++ 中 `constexpr` 的用法与作用

#### **核心作用 (Purpose)**

`constexpr` 的核心作用是将计算尽可能地从**运行时 (runtime)** 提前到**编译时 (compile time)**. 这样做带来了四大好处: 

1. **性能提升**: 编译时完成的计算, 程序运行时无需再做, 直接使用结果, 从而提升了执行效率. 
2. **编译期检查**: 可以在编译阶段使用 `static_assert` 对计算结果进行验证, 将潜在的运行时错误转变为编译错误. 
3. **常量保证**: `constexpr` 变量是真正的编译期常量, 可以用于数组大小、模板参数等必须在编译时确定的场景. 
4. **增强元编程**: 使得在编译期间执行更复杂的算法成为可能. 

#### **主要用法 (Usage)**

`constexpr` 可以修饰变量、函数和构造函数: 

- **`constexpr` 变量**: 必须在声明时用一个“常量表达式”来初始化. 它天生就是 `const` 的. 

- **`constexpr` 函数**: 是一种“两用函数”: 

  - 当传入的参数都是**编译期常量**时, 它就在**编译时**执行. 
  - 当传入的参数包含**运行时变量**时, 它就和普通函数一样在**运行时**执行. 
  - 可以把 lambda 表达式声明为`constexpr`: 
    ```cpp
    auto infDist = [](int val) constexpr -> int {...}
    ```

- **与 `const` 的关键区别**: 

  - `const` 强调**不变性 (Immutability)**: 变量初始化后值不能再改变, 但初始化可以在运行时进行. 
  - `constexpr` 强调**编译期可知性 (Compile-time Knowability)**: 变量的值必须在编译时就能确定. 

#### **规则（随 C++ 标准演进）**

- **C++11**：非常严格。函数体内只能包含一条 `return` 语句，不能有局部变量、循环、if/else（可以用三元运算符 `?:` 替代）等。
- **C++14**：限制被大大放宽。函数体内可以包含局部变量、循环 (`for`, `while`)、条件分支 (`if`, `switch`) 等，使其几乎和普通函数一样灵活。
- **C++17 及以后**：进一步放宽，例如允许在 `constexpr` 函数中使用 lambda 表达式。

```cpp
// C++11 风格 (使用递归)
constexpr long long factorial_cpp11(int n) {
    return n <= 1 ? 1 : (n * factorial_cpp11(n - 1));
}

// C++14 风格 (可以使用循环)
constexpr long long factorial_cpp14(int n) {
    long long result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

void usage_example() {
    // 编译时计算
    constexpr long long f5 = factorial_cpp14(5); // 结果 120 在编译时就算好了
    static_assert(f5 == 120, "Calculation error!"); // 可以在编译时断言

    int arr[f5]; // OK, f5是常量表达式，可以用来定义数组大小

    // 运行时计算
    int x;
    std::cin >> x;
    long long fx = factorial_cpp14(x); // x是运行时变量，函数在运行时执行
    std::cout << "Factorial of " << x << " is " << fx << std::endl;
}
```



### 2. `constexpr` 在编译期的执行者与执行方式

#### **由谁执行？(Who)**

`constexpr` 函数在编译期的执行者是 **C++ 编译器本身 (the C++ compiler)**. 

现代编译器内部集成了一个 C++ 的解释器或虚拟机, 专门用于在编译过程中执行这类计算任务. 

#### **如何执行？(How)**

这个过程可以概括为**“触发、执行、替换”**三部曲: 

1. **触发 (Trigger)**: 当编译器在代码中遇到一个**必须是常量表达式**的上下文时（例如初始化 `constexpr` 变量、`static_assert` 的条件、数组维度等）, 就会触发对 `constexpr` 函数的编译期求值. 
2. **执行 (Execution)**: 一旦触发, 编译器在其内部的一个高度受限的**“沙盒”环境**中模拟执行该函数. 这个执行环境非常特殊，可以理解为一个高度受限的“沙盒”：
   - **纯计算环境**：它只能访问在编译期已知的数据，比如传入的常量参数（如 `5`）和函数内部的计算。
   - **没有外部依赖**：它完全与外部世界隔离。不能进行任何 I/O 操作（如 `cout`, `cin`）、不能读写文件、不能进行动态内存分配 (`new`, `delete`)、不能调用系统 API、不能产生随机数等。
   - **确定性**：对于相同的输入，结果必须永远相同。这就是为什么所有依赖运行时状态的操作都被禁止的原因。
3. **替换 (Replacement)**: 函数执行完毕后, 编译器会得到一个确定的常量结果. 然后, 它会用这个**结果值**直接**替换**掉源代码中原来调用函数的那段表达式. 最终生成的可执行代码中, 将不包含这个函数调用, 只有一个硬编码的常量. 

例如, `int arr[factorial(5)];` 在编译后, 效果等同于 `int arr[120];`. 
