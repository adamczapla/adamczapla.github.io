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

## Iteration with Index: Accessing Elements and Positions

As already shown in `tuple_for_each`, iterating over a `std::tuple` requires a **parameter pack**. This is achieved by combining `std::apply` with a **lambda function using variadic template parameters**:

```c++
template <typename tuple_t, std::equality_comparable value_t>
constexpr auto tuple_find(tuple_t&& tuple, value_t const& value, size_t index = 0) noexcept {
  ...
  return std::apply([&](auto&&... tuple_values) {
    ...
  }, std::forward<tuple_t>(tuple));
  ...
}
```

> **Note:** This technique for unpacking a tuple using `std::apply` and variadic lambdas is explained in detail in [Part 1 of this series](https://adamczapla.github.io/posts/implementing-constexpr-compatible-algorithms-for-heterogeneous-containers-tuple-for-each/).
  {: .prompt-info }

However, for `tuple_find`, accessing the elements alone is not enough — we also need to know the **position** of each element within the tuple. 

This is essential to **resume the search** from a given point or to **return the position** of a match.

To achieve this, we use `std::make_index_sequence<N>`, where `N` is the number of elements in the tuple:

```c++
std::make_index_sequence<std::tuple_size<std::remove_cvref_t<tuple_t>>{}>{}
```

This creates an object of type `std::index_sequence<0, 1, 2, ..., N-1>` at compile time. We then pass it to a **lambda function with a deducible variadic template parameter** for the indices:

```c++
[&]<size_t... idx>(std::index_sequence<idx...>) {
  // Access to the indices: idx...
}
```

The indices `idx...` are **automatically deduced** from the `std::index_sequence` object during the call. Inside the lambda, they are available as a **parameter pack** — synchronized with the parameter pack of the tuple values.

This creates an **explicit mapping** between each individual **tuple element and its position** - the foundation for further processing:

```c++
template <typename tuple_t, std::equality_comparable value_t>
constexpr auto tuple_find(tuple_t&& tuple, value_t const& value, size_t index = 0) noexcept {
  return [&]<size_t... idx>(std::index_sequence<idx...>) {
    return std::apply([&](auto&&... tuple_values) {
      ...
    }, std::forward<tuple_t>(tuple));
  }(std::make_index_sequence<std::tuple_size<std::remove_cvref_t<tuple_t>>{}>{});
}
```

## Ensuring Reference Validity

Since `tuple_find` returns a **reference to the found element**, it must be ensured that this reference remains valid after the function call. A `static_assert` checks that all elements of the given tuple are **Lvalue references**:

```c++
static_assert(((std::is_lvalue_reference_v<decltype(tuple_values)>) && ...),
  "Error: All tuple elements must be lvalue references to ensure that returned "
  "references remain valid.");
```

This prevents returning references to **temporary objects**. A typical example of such an error is caught at **compile time**:

```c++
auto& result = tuple_find(std::tuple{5.5, 10, "str", 20, 'c', 5, 10, 10.0}, 10);
```

In this case, `result` **would be a dangling reference**, since the temporary `std::tuple` — and all its elements — are destroyed immediately after the function call.
