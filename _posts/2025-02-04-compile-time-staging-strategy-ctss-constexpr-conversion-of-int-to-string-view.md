---
title: "Compile-Time Staging Strategy (CTSS): constexpr Conversion of 'int to std::string_view'"
categories: [C++, Compile-time programming]
tags: [c++, c++20, c++23, constexpr, consteval, compile-time, int, string-view, array, literal-type, lambda, conversion]
description: "A detailed introduction to the Compile-Time Staging Strategy (CTSS) for converting an int to a std::string_view at compile time using C++20 and C++23."
---

## Introduction

With the latest `constexpr` extensions in C++, we can perform more and more computations at compile time. However, in practice, we repeatedly encounter a specific problem: **What do we do when a non-`constexpr` variable suddenly needs to be `constexpr` later in the program?**

**A typical example:** The `size` member of a `std::string` is used to instantiate a `std::array` of the corresponding size. The compiler rejects this because the size is **not** a constant expression.

This is where the **Compile-Time Staging Strategy (CTSS)** comes into play. It allows us to make values `constexpr` in multiple stages.

In this article, we will take a detailed look at how **CTSS** works, using the example of converting an `int` to a `std::string_view` at compile time.

## How Non-`constexpr` Values Can Become a Roadblock

**When converting an integer value to a string representation, we face a problem:** The number of required characters depends on the value itself. It is impossible to predict the number of digits in advance when the number is the result of a complex computation. Therefore, we need to create an oversized array and trim it to the exact required size.

```c++
namespace rng = std::ranges;

constexpr auto calculation(int init) { return (init % 17) * 42 + 5; }

template <auto buffer_size, auto int_builder>
consteval auto int_to_string_view() {
  std::array<char, buffer_size> oversized_buffer{};
  auto const result = std::to_chars(rng::begin(oversized_buffer),
                                    rng::end(oversized_buffer),
                                    int_builder());
  auto const right_size = rng::distance(rng::cbegin(oversized_buffer), result.ptr);
  std::array<char, right_size + 1> rightsize_buffer{}; // Error
  ...
}
```
