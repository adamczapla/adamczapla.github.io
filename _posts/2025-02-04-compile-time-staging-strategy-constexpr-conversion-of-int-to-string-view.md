---
title: "Compile-Time Staging Strategy (CTSS): constexpr Conversion of int to std::string_view"
date: 02-04-2025 12:00:00 +0100
categories: [C++, Compile-time programming]
tags: [c++, c++20, c++23, constexpr, consteval, compile-time, int, string-view, array, literal-type, lambda, conversion]
description: "A detailed introduction to the Compile-Time Staging Strategy (CTSS) for converting an int to a std::string_view at compile time using C++20 and C++23."
---

## Introduction

With the latest constexpr extensions in C++, we can perform more and more computations at compile time. However, in practice, we repeatedly encounter a specific problem: What do we do when a non-constexpr variable suddenly needs to be constexpr later in the program?

A typical example: The size member of a std::string is used to instantiate a std::array of the corresponding size. The compiler rejects this because the size is not a constant expression.

This is where the Compile-Time Staging Strategy (CTSS) comes into play. It allows us to make values constexpr in multiple stages.

In this article, we will take a detailed look at how CTSS works, using the example of converting an int to a std::string_view at compile time.

## Conclusion
