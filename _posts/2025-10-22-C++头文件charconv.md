---
layout: post
title: C++头文件 <charconv>
subtitle: 提供高性能, 无异常, 无内存分配, 且区域设置(locale)无关的基本数值类型与字符串之间的相互转换
author: Mikami
categories: cpp
tags: cpp code
---

`<charconv>` 是 C++17 标准库中引入的一个非常重要且实用的头文件. 它的核心目标是提供**高性能, 无异常, 无内存分配, 且区域设置(locale)无关**的**基本数值类型与字符串之间的相互转换**. 

在 `<charconv>` 出现之前, C++ 开发者通常有以下几种选择, 但它们各有缺点: 

1. **C 风格 (stdio):** `sprintf`, `sscanf`, `atoi`, `atof` 等. 
   - **缺点:** 类型不安全, 容易导致缓冲区溢出 (尤其是 `sprintf`) , 并且 `sscanf` 的性能不佳. 
2. **C++ 字符串流 (sstream):** `std::stringstream`. 
   - **缺点:** 性能开销大, 涉及动态内存分配, 并且受本地化 (locale) 影响 (例如, 某些地区用 `,` 作小数点) . 
3. **C++11 (string):** `std::stoi`, `std::stod`, `std::to_string` 等. 
   - **缺点:**
     - 它们会**抛出异常** ( `std::invalid_argument`, `std::out_of_range` ), 这在性能敏感的代码中 (如游戏循环, 高频交易) 可能是不可接受的开销. 
     - `std::to_string` 内部**可能进行内存分配**. 
     - 它们仍然受本地化 (locale) 影响. 

`<charconv>` 旨在彻底解决以上所有痛点. 

------

### `<charconv>` 的核心特性

1. 高性能 (High Performance):

   它的实现被高度优化, 通常是 C++ 中最快的数字-字符串转换方式, 甚至快于 C 风格的 itoa (非标准) 或 sprintf. 

2. 无内存分配 (Non-allocating):

   所有操作都在用户提供的固定大小的字符缓冲区上进行. 它不会在堆上分配任何内存. 

3. 无异常 (Non-throwing):

   它通过返回一个包含错误码的 struct 来报告成功或失败, 而不是抛出异常. 这使得错误处理的开销极低. 

4. 区域设置无关 (Locale-independent):

   它始终使用 "C" locale 规则 (例如, 小数点总是 '.') . 这对于配置文件, JSON, XML, 网络协议等需要固定格式的数据序列化和反序列化至关重要. 

------

### 两个核心函数

`<charconv>` 只提供了两个核心函数模板 (及其重载) : 

1. **`std::to_chars`**: 将数值转换为字符串. 
2. **`std::from_chars`**: 将字符串转换为数值. 

它们都通过返回一个 `struct` 来报告操作结果. 

------

### 1. `std::to_chars` (数值 -> 字符串)

它尝试将一个数值格式化后写入你提供的字符缓冲区 `[first, last)` 中. 

#### 函数签名 (整数)

```cpp
std::to_chars_result std::to_chars(
    char* first,          // 缓冲区的起始指针
    char* last,           // 缓冲区的末尾指针 (one-past-the-end)
    T value,              // 要转换的数值
    int base = 10         // 进制 (支持 2 到 36)
);
```

- **`T`**: 可以是 `int`, `long`, `unsigned int`, `unsigned long long` 等各种整数类型. 

#### 返回值: `std::to_chars_result`

这是一个 `struct`, 定义大致如下: 

```cpp
struct std::to_chars_result {
    char* ptr;          // 指向写入的最后一个字符的 *下一个* 位置
    std::errc ec;       // 错误码
};
```

- 如果 `ec` 是 `std::errc()` (即默认构造, 表示成功), 则 `[first, ptr)` 范围内就是转换后的字符串. 
- 如果 `ec` 是 `std::errc::value_too_large`, 表示你提供的缓冲区 `[first, last)` 不够大, 无法容纳转换后的字符串. 

------

### 2. `std::from_chars` (字符串 -> 数值)

它尝试从字符缓冲区 `[first, last)` 中解析一个数值. 

#### 函数签名 (整数)

```cpp
std::from_chars_result std::from_chars(
    const char* first,    // 要解析的字符串的起始指针
    const char* last,     // 要解析的字符串的末尾指针
    T& value,             // [out] 用于接收解析结果的变量
    int base = 10         // 进制 (支持 2 到 36)
);
```

- **`T`**: 必须是一个整数类型. 

#### 返回值: `std::from_chars_result`

定义大致如下: 

```cpp
struct std::from_chars_result {
    const char* ptr;    // 指向 *未被* 解析的第一个字符
    std::errc ec;       // 错误码
};
```

- 如果 `ec` 是 `std::errc()` (成功): 
  - `value` 中存储了T-sl" >解析到的值. 
  - `ptr` 指向第一个无法解析的字符. 例如, 解析 "12345abc", `ptr` 会指向 'a'. 
- 如果 `ec` 是 `std::errc::invalid_argument`: 
  - 表示在 `[first, last)` 范围内找不到任何有效的数字模式. 
- 如果 `ec` 是 `std::errc::result_out_of_range`: 
  - 表示解析到的数字超出了类型 `T` 所能表示的范围 (上溢或下溢) . 

------

### 浮点数支持

`std::to_chars` 和 `std::from_chars` 同样重载了 `float`, `double` 和 `long double`. 

浮点数版本的函数签名略有不同, 它们接受一个 `std::chars_format` 枚举来指定格式 (如科学计数法, 固定小数点等) , 并且 (目前) 不支持指定 `base` (总是 10 进制) . 

```cpp
// 浮点数 to_chars 示例
std::to_chars_result std::to_chars(
    char* first, char* last,
    double value,
    std::chars_format fmt = std::chars_format::general,
    int precision = -1 // 默认精度
);

// 浮点数 from_chars 示例
std::from_chars_result std::from_chars(
    const char* first, const char* last,
    double& value,
    std::chars_format fmt = std::chars_format::general
);
```

`std::chars_format` 的值包括: 

- `std::chars_format::general` (默认, 结合了 fixed 和 scientific)
- `std::chars_format::fixed` (定点表示)
- `std::chars_format::scientific` (科学计数法)
- `std::chars_format::hex` (十六进制浮点数)

------

### 完整示例代码

下面是一个结合使用 C++17 的 `std::string_view` 和结构化绑定 (Structured Bindings) 的现代 C++ 示例: 

```cpp
#include <iostream>
#include <charconv>     // 核心头文件
#include <string>
#include <string_view>
#include <system_error> // std::errc
#include <array>        // std::array

// 辅助函数: 将 std::errc 转换为字符串以便打印
std::string error_to_string(std::errc ec) {
    if (ec == std::errc()) {
        return "Success";
    }
    return std::make_error_code(ec).message();
}

int main() {
    // ------------------------------------------------
    // 示例 1: std::to_chars (int -> string)
    // ------------------------------------------------
    std::cout << "--- std::to_chars ---" << std::endl;
    int my_value = 1989;
    
    // 必须提供一个缓冲区. std::array 是一个很好的选择. 
    // 整数最多约20位 (64位) + 负号, 30 足够了. 
    std::array<char, 30> buffer; 

    // 使用 C++17 结构化绑定
    if (auto [ptr, ec] = std::to_chars(buffer.data(), buffer.data() + buffer.size(), my_value, 10);
        ec == std::errc()) 
    {
        // 成功！
        // ptr 指向写入的末尾
        // 使用 string_view 来安全地封装结果, 无需拷贝
        std::string_view sv(buffer.data(), ptr - buffer.data());
        
        std::cout << "Value: " << my_value << std::endl;
        std::cout << "String: '" << sv << "'" << std::endl;
        std::cout << "Length: " << sv.length() << std::endl;
    } else {
        // 缓冲区太小 (value_too_large) 或其他错误
        std::cout << "Conversion failed: " << error_to_string(ec) << std::endl;
    }

    // ------------------------------------------------
    // 示例 2: std::from_chars (string -> int)
    // ------------------------------------------------
    std::cout << "\n--- std::from_chars ---" << std::endl;
    std::string_view str_valid = "42 Hello";
    std::string_view str_invalid = "NotANumber";
    std::string_view str_overflow = "99999999999999999999999999999";
    int result = 0;

    // 尝试解析 "42 Hello"
    if (auto [ptr, ec] = std::from_chars(str_valid.data(), str_valid.data() + str_valid.size(), result, 10);
        ec == std::errc())
    {
        std::cout << "Parsed '" << str_valid << "' -> " << result << std::endl;
        std::cout << "Remaining string starts at: '" << ptr << "'" << std::endl; // 应该指向 ' Hello'
    } else {
        std::cout << "Failed to parse '" << str_valid << "': " << error_to_string(ec) << std::endl;
    }

    // 尝试解析 "NotANumber"
    if (auto [ptr, ec] = std::from_chars(str_invalid.data(), str_invalid.data() + str_invalid.size(), result, 10);
        ec == std::errc())
    {
        std::cout << "Parsed '" << str_invalid << "' -> " << result << std::endl;
    } else {
        // 这将失败, ec == std::errc::invalid_argument
        std::cout << "Failed to parse '" << str_invalid << "': " << error_to_string(ec) << std::endl;
    }

    // 尝试解析溢出的数字
    if (auto [ptr, ec] = std::from_chars(str_overflow.data(), str_overflow.data() + str_overflow.size(), result, 10);
        ec == std::errc())
    {
        std::cout << "Parsed '" << str_overflow << "' -> " << result << std::endl;
    } else {
        // 这将失败, ec == std::errc::result_out_of_range
        std::cout << "Failed to parse '" << str_overflow << "': " << error_to_string(ec) << std::endl;
    }
    
    return 0;
}
```

输出如下: 

```
--- std::to_chars ---
Value: 1989
String: '1989'
Length: 4

--- std::from_chars ---
Parsed '42 Hello' -> 42
Remaining string starts at: ' Hello'
Failed to parse 'NotANumber': Invalid argument
Failed to parse '99999999999999999999999999999': Result too large
```

### 总结

`<charconv>` 是 C++17 中一项革命性的补充. 当你需要进行**高性能**的数字和字符串转换时, 它应该是你的**首选**, 特别是在以下场景: 

- 解析 JSON, XML 或自定义的配置文件. 
- 网络编程中序列化/反序列化消息. 
- 游戏引擎或实时模拟. 
- 任何你希望避免异常和内存分配的底层库代码. 
