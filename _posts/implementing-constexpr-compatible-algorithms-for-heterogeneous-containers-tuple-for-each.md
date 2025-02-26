---
title: "Implementing constexpr-Compatible Algorithms for Heterogeneous Containers – Part 1: 'tuple_for_each'"
categories: [Algorithms for Heterogeneous Containers, Compile-time programming]
tags: [c++, constexpr, compile-time, lambda, nttp, tuple, algorithms, heterogeneous-containers, tuple-for-each, std-apply, perfect-forwarding, fold-expressions, parameter-pack, variadic-template, forwarding-reference, comma-operator, if-constexpr]
description: "..."
---

## Introduction

Heterogeneous containers are an essential tool in modern C++ applications, especially when values of different types need to be managed and processed together. While the standard library provides numerous algorithms for **homogeneous containers** such as `std::vector`, similar functionality for **heterogeneous containers** like `std::tuple` is missing.

This article series aims to close this gap - not by introducing a ready-made library, but by demonstrating **how to develop such algorithms** from scratch. The goal is to remove barriers that prevent many developers from writing their own algorithms for heterogeneous containers.

The idea of treating `std::tuple` as a fully-fledged container and developing algorithms for it emerged during the implementation of a **compile-time command-line parser**, where `std::tuple` proved to be extremely useful.

This article focuses on implementing one such algorithm: **tuple_for_each**. This function enables iteration over each element in an `std::tuple` and applies a given action to each element - similar to `std::for_each` for homogeneous containers.

Beyond explaining a useful function, the article aims to provide a deeper understanding of the mechanisms involved. Once these principles are understood, developing custom algorithms for `std::tuple` or other heterogeneous containers - whether for iteration, search, or transformation - becomes significantly easier.

## tuple_for_each – Iteration Over Heterogeneous Containers

We start with the simplest operation: **iterating over an `std::tuple`**. This first algorithm serves not only as a practical application but also introduces fundamental techniques necessary for developing further algorithms for heterogeneous containers.

To illustrate how tuple_for_each works, consider the following simple example:

code

Here, tuple_for_each invokes the provided action on each element within the `std::tuple`, similar to `std::for_each` for homogeneous containers.

> **Note:** tuple_for_each is **not limited to runtime execution**; it is specifically designed to be used in compile-time programming as well.
  {: .prompt-info }

## Passing the Tuple and Perfect Forwarding

The heterogeneous container is passed as a **forwarding reference** (`tuple_t&&`), allowing **perfect forwarding** to preserve the **value category** of both the `std::tuple` and its elements.

code

If the original `std::tuple` was **modifiable**, it remains so within the action. Conversely, if the original `std::tuple` was **immutable**, that restriction is also enforced within the action.

**This flexibility ensures that the function supports a wide range of use cases without introducing unnecessary constraints.**

The action is **passed by value** since it is typically a **stateless lambda** or **functor**, which requires only **1 byte** of storage. Passing it by reference would generally be less efficient, consuming at least **4 or 8 bytes**.




















