---
title: "Implementing 'tuple_find' – A Constexpr-Compatible Algorithm for Heterogeneous Containers - Part 2"
categories: [Compile-time programming, Algorithms for Heterogeneous Containers]
tags: [c++, constexpr, compile-time, lambda, tuple, std-tuple, algorithms, heterogeneous-containers, tuple-for-each, tuple-find, std-apply, fold-expressions, parameter-pack, variadic-template, comma-operator, value-category, static_assert]
description: "A detailed guide to implementing the constexpr-compatible search algorithm tuple_find for heterogeneous containers using modern C++ - Part 2 of the Series."
---

## Introduction

In the [first part](https://adamczapla.github.io/posts/implementing-constexpr-compatible-algorithms-for-heterogeneous-containers-tuple-for-each/) of this article series, we introduced `tuple_for_each` – an algorithm that makes it possible to iterate over all elements of a `std::tuple` and apply an action to each one, without using traditional loops or iterators. Based on that foundation, this second article covers another common use case for heterogeneous containers: **searching for a specific value**.

In standard containers like `std::vector`, this task is typically solved using std::find or `std::ranges::find`. For `std::tuple`, however, no such equivalent exists. This article **closes that gap** by explaining step by step how such a function can be implemented.

The algorithm `tuple_find` looks for the **first element** in the tuple that matches a given value. It returns a **reference** to that element along with its **position**. The returned position allows the search to continue at that exact point in a later call, so additional matches can be found.

At the same time, it takes into account whether the found element is a `const` or a **modifiable reference**. To represent this difference, the result is wrapped in a `std::variant`. Combined with `std::optional`, this also indicates whether a **matching element** was found at all.

Unlike standard search functions, `tuple_find` must handle some specific challenges related to **type safety** and **reference semantics** in C++. These challenges and the resulting **design decisions** are the main focus of this article.
