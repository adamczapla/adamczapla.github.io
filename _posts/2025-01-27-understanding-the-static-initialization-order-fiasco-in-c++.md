---
title: "Understanding the 'Static Initialization Order Fiasco' in C++"
date: 27-01-2025 12:00:00 +0100
categories: [C++, Compile-time programming]
tags: [c++, static-initialization, initialization-order, constexpr, constinit, global-variables, static-variables, compile-time, linker, zero-initialization]
description: A concise exploration of the Static Initialization Order Fiasco in C++, including its causes, examples, and solutions for developers.
image: /assets/images/solution.png
---

## Introduction

The **Static Initialization Order Fiasco** is a critical issue in C++ programming that can lead to unpredictable behavior in global and static variables. For C++ developers, understanding this phenomenon is essential. This post explores the problem, illustrates it with examples, and provides solutions.

## The Problem

**Global** and **static** variables in C++ are usually initialized at compile time if their values can be determined at that time. However, if the initialization depends on a function that is not `constexpr`, even if the function's arguments are known, compile-time initialization is not possible. In such cases, the variable is set to `0` (`zero-initialized`[^1]). 

Let’s look at an example to understand this issue better.

### `file1.cpp`

```c++
auto sum(int a, int b) {
  return a + b;
}

int sum_result = sum(5, 5);
```

The global variable `sum_result` is set to `0` (`zero-initialized`) because the `sum` function is not marked as `constexpr`. Even though the values `5` and `5` are known at compile time, the lack of `constexpr` prevents the compiler from evaluating the result at compile time. Moreover, even with `constexpr`, initialization is not guaranteed unless stricter requirements like `consteval` are used — this will be discussed later.

### `file2.cpp`

```c++
extern int sum_result;
int static_val = sum_result;
```

The global variable `static_val` is also set to `0` (`zero-initialized`), as the value of `sum_result` is defined in another **translation unit** and not available in the current one.

### `main.cpp`

```c++
extern int sum_result;
extern int static_val;

auto main() -> int {
  std::cout << "sum_result = " << sum_result << '\n'; // output: 10
  std::cout << "static_val = " << static_val << '\n'; // output: 10 oder 0
  return 0;
}
```

The **linker** decides at link time which **translation unit** it processes first.
- If it processes `file1.cpp` first, `sum_result` is initialized to `10`, and `static_val` is also set to `10`.
- If it processes `file2.cpp` first, `static_val` is set to `0`, and then `sum_result` is initialized to `10`.

This behavior leads to a discrepancy between the values of `sum_result` and `static_val`, even though they are logically expected to be the same. 

> This `inconsistency` is the essence of the Static Initialization Order Fiasco.
{: .prompt-info }

## The Solution

### `file1.cpp`

```c++
constexpr auto sum(int a, int b) {
  return a + b;
}

constinit int sum_result = sum(5, 5);
```

Here, `sum_result` is initialized at compile time because the `sum` function is `constexpr`, and the values (`5` and `5`) are known at compile time. To enforce compile-time initialization, the `constinit` keyword is **required**.

### `file2.cpp`

```c++
extern constinit int sum_result; 
int static_val = sum_result;
```

Because `sum_result` is defined in another translation unit, `static_val` is `zero-initialized`[^1] since the value of `sum_result` is not available during the compile step.

### `main.cpp`

```c++
extern constinit int sum_result;
extern int static_val;

auto main() -> int {
  std::cout << "sum_result = " << sum_result << '\n'; // output: 10
  std::cout << "static_val = " << static_val << '\n'; // output: 10
  return 0;
}
```

Now, it no longer matters which translation unit the linker processes first. `static_val` will **always** be initialized with the value `10` because `sum_result` is **guaranteed** to be initialized at compile time.

## Conclusion

To guarantee initialization of **global** or **static** variables at compile time:

`1.` All values must be known at compile time

`2.` The variable must be marked with either the `constexpr` or `constinit` keyword

Although compilers often perform compile-time initialization without these keywords, this behavior is **not guaranteed** unless the keywords are **explicitly** used.

Understanding the **Static Initialization Order Fiasco** not only helps avoid potential bugs but also highlights the importance of **explicit** initialization in C++ programming. By leveraging C++ features like `constexpr` and `constinit`, developers can ensure consistency and predictability in their applications.

## Share your feedback

### Praise or criticism is appreciated!

<script src="https://giscus.app/client.js"
        data-repo="adamczapla/adamczapla.github.io"
        data-repo-id="R_kgDONv6EUg"
        data-category="Announcements"
        data-category-id="DIC_kwDONv6EUs4CmqH2"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="en"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>

## Footnote

[^1]: `Zero-initialization` depends on the data type and sets the value to a `null value` defined by the type:
    - For arithmetic types (e.g., `int`, `float`), it is `0` or `0.0`.
    - For pointers, it is `nullptr`.
    - For `bool`, it is `false`.
    - For characters (`char`), it is `'\0'`.
    - For user-defined types (e.g., `classes/structs`), all members are recursively `zero-initialized`.
