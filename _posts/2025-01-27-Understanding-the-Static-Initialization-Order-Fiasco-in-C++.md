---
title: Understanding the Static Initialization Order Fiasco in C++
categories: [Programming, C++]
tags: [c++, static-initialization, initialization-order, constexpr, constinit, global-variables, static-variables, compile-time, linker, zero-initialization]
description: A concise exploration of the Static Initialization Order Fiasco in C++
---

## Introduction

The Static Initialization Order Fiasco is a critical issue in C++ programming that can lead to unpredictable behavior in global and static variables. For C++ developers, understanding this phenomenon is essential. This document explores the problem, illustrates it with examples, and provides solutions.

## The Problem

Global and static variables in C++ are usually initialized at compile time if their values are known at that time. If the value for initializing a variable is not known at compile time, the variable is set to 0 (*`zero initialized`). However, this initialization is **not always** guaranteed. Let’s look at an example to understand this issue better. 

### `file1.cpp`

code

The global variable `sum_result` is set to 0 (*`zero initialized`) because the `sum` function is not `constexpr`. Compile-time initialization is therefore not possible.

### `file2.cpp`

code

The global variable `static_val` is also set to 0 (*`zero initialized`), as the value of `sum_result` is defined in another **translation unit** and not available in the current one.

### `main.cpp`

code

The **linker** decides at link time which **translation unit** it processes first.
- If it processes `file1.cpp` first, `sum_result` is initialized to 10, and `static_val` is also set to 10.
- If it processes `file2.cpp` first, `static_val` is set to 0, and then `sum_result` is initialized to 10.

This behavior leads to a discrepancy between the values of `sum_result` and `static_val`, even though they are logically expected to be the same. 

> This `inconsistency` is the essence of the Static Initialization Order Fiasco.
{: .prompt-info }

## The Solution

### `file1.cpp`

code

Here, `sum_result` is initialized at compile time because the `sum` function is `constexpr`, and the values (5 and 5) are known at compile time. To enforce compile-time initialization, the `constinit` keyword **is required**.

### `file2.cpp`

code

Because `sum_result` is defined in another translation unit, `static_val` is zero-initialized since the value of `sum_result` is not available during the compile step.

### `main.cpp`

code

Now, it no longer matters which translation unit the linker processes first. `static_val` **will always** be initialized with the value 10 because `sum_result` **is guaranteed** to be initialized at compile time.

## Conclusion

To guarantee initialization of global or static variables at compile time:

`1.` All values must be known at compile time

`2.` The variable must be marked with either the `constexpr` or `constinit` keyword

Although compilers often perform compile-time initialization without these keywords, this behavior **is not guaranteed** unless the keywords are **explicitly** used.

Understanding the ***Static Initialization Order Fiasco*** not only helps avoid potential bugs but also highlights the **importance of explicit initialization** in C++ programming. By leveraging C++ features like `constexpr` and `constinit`, developers can ensure consistency and predictability in their applications.

Footnote
* `Zero-initialization` depends on the data type and sets the value to a "null value" defined by the type:
- For arithmetic types (e.g., int, float), it is 0 or 0.0.
- For pointers, it is nullptr.
- For bool, it is false.
- For characters (char), it is '\0'.
- For user-defined types (e.g., classes/structs), all members are recursively zero-initialized.*



```c++
constexpr auto is_constant_evaluated(int val) {
  if (std::is_constant_evaluated()) {
    return consteval_func(val);
  } else {
    std::cout << "is_constant_evaluated==false\n"; return 0;
  }
}
```
