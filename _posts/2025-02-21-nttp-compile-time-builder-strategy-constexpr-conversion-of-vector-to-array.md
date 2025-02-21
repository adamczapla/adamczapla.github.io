---
title: "NTTP Compile-Time Builder Strategy - constexpr Conversion of 'std::vector to std::array'"
categories: [C++, Compile-time programming]
tags: [c++, c++20, constexpr, compile-time, int, array, vector, literal-type, lambda, conversion, staging, strategy, nttp, constant-expression]
description: "A deep dive into the 'NTTP Compile-Time Builder Strategy' for constexpr conversion of 'std::vector' to 'std::array' in C++."
---

## Introduction

**Non-Type Template Parameters (NTTPs)** are a powerful feature of compile-time programming in C++. They allow values to be passed and processed at compile time. However, they come with a significant limitation: **only literal types are allowed**.

This means that many commonly used standard types, such as `std::string` or `std::vector`, **cannot be passed as NTTPs**. In many use cases, it would be beneficial to utilize dynamic or complex data types in a similar manner.

The **NTTP Compile-Time Builder Strategy** provides a way to achieve exactly that.

This article explains how this technique works and in which scenarios it can be effectively used.

## Core Concept: The NTTP Compile-Time Builder Strategy

The NTTP Compile-Time Builder Strategy consists of the following steps:

1. **Encapsulation of Value Creation in a Builder**

Instead of passing the value directly, we define a `constexpr` lambda function that generates and returns the desired value. This lambda acts as a **builder**, describing **how the value is created** without instantiating it immediately.

code

2. **Passing the Builder as an NTTP**

Since lambdas in C++ are implicitly `constexpr`, they can be passed as NTTPs. This allows non-literal types to be processed at compile time indirectly.

code

3. **Generating the Value within the Function**

Inside the target function (`foo`), the builder is executed to generate the desired value at compile time.

code

After explaining the fundamentals of the NTTP Compile-Time Builder Strategy, let’s look at a concrete use case.

## Practical Example: Converting `std::vector<int>` to `std::array<int, N>`

### Implementation:

code

### How Does the Conversion Work?

The to_array function applies the **NTTP Compile-Time Builder Strategy** to transform a `std::vector<int>` into a `std::array<int, N>` at compile time.

The first NTTP defines the size of the array. This size is not automatically derived but must be explicitly passed at the function call. It must be large enough to hold the entire contents of the `std::vector`. The size cannot be inferred from `std::vector` because `std::array` requires a constant expression for its size.

The second NTTP is the **builder**, which describes how the `std::vector` is created.

Inside `to_array`, the builder executes to generate a `std::vector` at compile time.

The contents of the vector are copied into an **oversized array** (`std::array<int, max_size>`), and the actual number of inserted elements is determined.

These values—the temporary array and its actual size—are returned in a `std::pair`.

### Why This Intermediate Step?

The key limitation of `std::array` is that its **size must be known at compile time**. We can only create the final array once its exact size is available as a constant expression. To achieve this, the entire process is **wrapped** in a lambda function, a technique known as **Compile-Time Staging Strategy (CTSS)**.

**-> For more details on CTSS:**
Compile-Time Staging Strategy (CTSS): `constexpr` Conversion of `int` to `std::string_view`

In the final step, the temporary oversized array is **trimmed to its actual size** and returned as a `constexpr` value.

### Usage Example:

code 

Now, the builder’s result is available as a `constexpr` value and can be further processed.












