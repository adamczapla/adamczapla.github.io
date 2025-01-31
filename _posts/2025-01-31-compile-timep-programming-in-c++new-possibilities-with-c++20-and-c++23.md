---
title: "Compile-Time Programming in C++: New Possibilities with C++20 and C++23"
date: 31-01-2025 12:00:00 +0100
categories: [C++, Compile time programming]
tags: [c++, c++20, c++23, constexpr, consteval, compile-time]
description: "A detailed exploration of ..."
---

## Introduction

Efficiency is a key factor in modern C++ projects. Especially in performance-critical applications, it is beneficial to avoid expensive memory allocations and runtime computations. The new features in C++20 and C++23 greatly expand compile-time programming, allowing even non-literal types like std::string and std::vector to be processed at compile time.

In this article, I will show a concrete example of how to efficiently convert std::string into std::string_view at compile time. This reduces runtime costs, avoids unnecessary dynamic memory allocations, and enables new optimizations – such as for logging or generated code metadata. In addition to the new language features, I will explain fundamental concepts of compile-time programming and present practical solutions to common challenges.

## Challenge: Using std::string as constexpr

Every call to a constexpr or consteval function from a non-constexpr context requires all function arguments to be literal types, meaning their values must be known at compile time.

However, std::string is not a literal type, even though it has constexpr constructors since C++20 and can be used in a constexpr context at compile time.

Let's consider the following example:

code

Here, the call fails because std::string cannot be used as an argument in a constexpr function to initialize a constexpr variable.

It also does not help to declare the variable str as constexpr, as shown in the following example:

code

### 1. Passing a prvalue instead of an lvalue

Instead of passing a variable, we use a temporary value (prvalue):

code

Since C++17, prvalues are no longer objects but pure expressions. The materialization into an object occurs only within the constexpr function to_string_view, making the code valid because the temporary std::string does not need to be constexpr at the time of the call.

### 2. Using a Lambda Function

Alternatively, the std::string can be encapsulated in a lambda:

code

Since lambdas have been implicitly constexpr since C++17, the call to the constexpr function takes place from a constexpr context.

## Converting std::string into std::array

To implement to_string_view, we first convert std::string into std::array. 

code

The key limitation of std::string and other non-literal types is that they must deallocate their memory in a constexpr context. If their values need to leave the constexpr context, they must be copied into a literal type.

std::array is therefore an ideal choice for storing the std::string value. The maximum size of the array is passed as a Non-Type Template Parameter (NTTP) because function parameters in C++ can never be constexpr and when instantiating the right_size_array array, max_size must be a constant expression.

### Dynamic Adjustment of Array Size

Since we never know the exact array size in advance, we first create an oversized array and then trim it to the exact size.

code

### Problem: Function Parameters in C++ Are Never constexpr

The function parameter str is not constexpr, yet we pass it as an argument to a constexpr function that initializes a constexpr variable. However, the variable intermediate_data must remain constexpr because, when instantiating the right_size_array array (in the next line), the size must be a constant expression.

### Solution: Lambda as NTTP

Instead of passing std::string as a parameter, we encapsulate it in a lambda and pass it as an NTTP.

code

### Optimization: Optional

To keep everything in place, we encapsulate the function to_oversized_array in a lambda.

code

## Converting std::array into std::string_view

By marking the array right_size_array with static constexpr, we store it in static memory and allow it to be referenced using a std::string_view instance. This instance is then returned to the caller of the to_string_view function.

code

### Portability: Using to_static for Clang

Since Clang has issues with static constexpr in consteval functions in C++23, the following helper function ensures portability:

code

We call this function with the array right_size_array as a Non-Type Template Parameter (NTTP). NTTPs allow values to be stored directly in the static memory area, making them referenceable. This way, we can safely store std::array data in static memory and return it as std::string_view.

code

With this, our compile-time conversion from std::string to std::string_view is not only complete but also portable and efficient.

## Use Case: Compile-Time Generation of Log Tags

A practical example is creating log tags for generic types:

code

Here, a log tag for a generic type is created at compile time. This reduces runtime costs and avoids unnecessary memory allocations.

## Conclusion

The new features in C++20/23 enable powerful compile-time manipulations even for non-literal types. The techniques shown allow efficient conversion of std::string into std::string_view, reducing runtime costs.

Especially in performance-critical applications, compile-time programming can provide significant advantages. The ability to process strings efficiently at compile time opens up exciting optimization possibilities – not only for logging but also for many other areas of modern C++ development.
