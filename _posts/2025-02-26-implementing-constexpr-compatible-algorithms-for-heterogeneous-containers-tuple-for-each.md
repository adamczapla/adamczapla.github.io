---
title: "Constexpr-Compatible Function for std::tuple - Part 1: tuple_for_each"
categories: [Compile-time programming, Algorithms for Heterogeneous Containers]
tags: [c++, constexpr, compile-time, lambda, nttp, tuple, std-tuple, algorithms, heterogeneous-containers, tuple-for-each, std-apply, perfect-forwarding, fold-expressions, parameter-pack, variadic-template, forwarding-reference, comma-operator, if-constexpr, std-invoke, value-category, static_assert]
description: "A comprehensive guide to implementing the constexpr-compatible algorithm 'tuple_for_each' for heterogeneous containers using modern C++."
image: /assets/images/strictmode.png
---

## Introduction

Heterogeneous containers are an essential tool in modern C++ applications, especially when values of different types need to be managed and processed together. While the standard library provides numerous algorithms for **homogeneous containers** such as `std::vector`, similar functionality for **heterogeneous containers** like `std::tuple` is missing.

This article series aims to close this gap - not by introducing a ready-made library, but by demonstrating **how to develop such algorithms** from scratch. The goal is to remove barriers that prevent many developers from writing their own algorithms for heterogeneous containers.

The idea of treating `std::tuple` as a fully-fledged container and developing algorithms for it emerged during the implementation of a **compile-time command-line parser**, where `std::tuple` proved to be extremely useful.

This article focuses on implementing one such algorithm: `tuple_for_each`. This function enables iteration over each element in an `std::tuple` and applies a given action to each element - similar to `std::for_each` for homogeneous containers.

Beyond explaining a useful function, the article aims to provide a deeper understanding of the mechanisms involved. Once these principles are understood, developing custom algorithms for `std::tuple` or other heterogeneous containers - whether for iteration, search, or transformation - becomes significantly easier.

## tuple_for_each – Iteration Over Heterogeneous Containers

We start with the simplest operation: **iterating over an** `std::tuple`. This first algorithm serves not only as a practical application but also introduces fundamental techniques necessary for developing further algorithms for heterogeneous containers.

To illustrate how `tuple_for_each` works, consider the following simple example:

```c++
std::tuple values{42, 3.14, "Hello"};
tuple_for_each(values, [](auto const& value) {
  std::cout << value << '\n';
});
```
**Output:**
```
42
3.14
Hello
```

Here, `tuple_for_each` invokes the provided action on each element within the `std::tuple`, similar to `std::for_each` for homogeneous containers.

> `tuple_for_each` is **not limited to runtime execution**; it is specifically designed to be used in compile-time programming as well.
  {: .prompt-info }

## Passing the Tuple and Perfect Forwarding

The heterogeneous container is passed as a **forwarding reference** (`tuple_t&&`), allowing **perfect forwarding** to preserve the **value category** of both the `std::tuple` and its elements.

```c++
template <typename tuple_t, typename action_t>
constexpr auto tuple_for_each(tuple_t&& tuple, action_t action) noexcept {
  ...
}
```

If the original `std::tuple` was **modifiable**, it remains so within the action. Conversely, if the original `std::tuple` was **immutable**, that restriction is also enforced within the action.

This flexibility ensures that the function supports a wide range of use cases without introducing unnecessary constraints.

The action is **passed by value** since it is typically a **stateless lambda** or **functor**, which requires only `1 byte` of storage. Passing it by reference would generally be less efficient, consuming at least `4 or 8 bytes`.

## Iterating Over Tuple Elements with `std::apply` and Fold Expressions

Heterogeneous containers like `std::tuple` do not support classical iteration with loops or iterators, unlike homogeneous containers such as std::vector. To iterate over all tuple values, they must be made available as a **parameter pack**.

To achieve this, the processing code is encapsulated in a **lambda function**, which is then forwarded along with the `std::tuple` to `std::apply`:

```c++
std::apply([](auto&&... tuple_values) {
  // Access to all elements in the tuple
}, std::forward<tuple_t>(tuple));
```

`std::apply` extracts the elements from the `std::tuple` and passes them as arguments to the lambda function. The **variadic template** with **forwarding references** (`auto&&... tuple_values`) ensures that the elements are available as a **parameter pack** for subsequent iteration.

The actual iteration over the elements is performed using a **fold expression** with the **comma operator**:

```c++
(call_action(std::forward<decltype(tuple_values)>(tuple_values)), ...);
```

This fold expression invokes `call_action` for each element in the parameter pack.

```c++
auto call_action = [&](auto&& tuple_value) {
  if constexpr (std::invocable<action_t, decltype(tuple_value)>) {
    std::invoke(action, std::forward<decltype(tuple_value)>(tuple_value));
  }
};
```

The `if constexpr` condition checks whether the action can be invoked with the current element (`std::invocable`). If the condition is met, `std::invoke` is used to execute the action on the tuple element.

**Perfect forwarding** (`std::forward`) ensures that the original **value category** of the element is maintained, avoiding unnecessary copies and preserving references.

### Why `if constexpr`?

A normal `if` statement would **not work** in this case because it is a **runtime construct**. This means the compiler would **instantiate all possible code paths** regardless of the condition. If even a single tuple element is incompatible with the action, the `std::invoke` call would fail, resulting in a compilation error.

`if constexpr`, on the other hand, is a **compile-time construct**. The compiler eliminates code paths where the condition evaluates to false, ensuring that only **valid elements are passed** to the action.

## Strict Mode – Ensuring Compatibility with All Elements

By default, `tuple_for_each` is designed to **skip elements that are not compatible** with the provided action. If the action cannot be invoked with a specific element type, that element is simply ignored.

However, in some cases, it may be required that the action be **compatible with every element** in the tuple. This is where the **strict mode** comes into play.

Strict mode is enabled using a **boolean non-type template parameter (NTTP)**:

```c++
template <bool strict = false, typename tuple_t, typename action_t>
constexpr auto tuple_for_each(tuple_t&& tuple, action_t action) noexcept {
  std::apply([&](auto&&... tuple_values) {
    static_assert(!strict || ((std::invocable<action_t, decltype(tuple_values)>) && ...),
      "Error: In strict mode, the action must be callable with every element in the tuple.");
    auto call_action = [&](auto&& tuple_value) {
      if constexpr (std::invocable<action_t, decltype(tuple_value)>) {
        std::invoke(action, std::forward<decltype(tuple_value)>(tuple_value));
      } 
    };
    (call_action(std::forward<decltype(tuple_values)>(tuple_values)), ...);    
  }, std::forward<tuple_t>(tuple));
}
```

### How It Works

* **strict = false (default behavior):** If an element is not compatible with the action, it is simply skipped. 
* **strict = true:** The action must be callable with every element in the tuple. If this requirement is not met, `static_assert` produces a compile-time error. 

### Why Is Strict Mode an NTTP?

Strict mode **must be configured at compile time** since it is evaluated inside `static_assert`. Therefore, it must be defined as a **non-type template parameter (NTTP)** rather than a function argument.

### Example: Enforcing Strict Mode

```c++
std::tuple values{42, 3.14, "Hello"};
tuple_for_each<true>(values, [](int value) {
  std::cout << value << '\n';
});
```

Since `"Hello"` (`char const[6]`) is incompatible with `int`, compilation fails.

## Conclusion

With `tuple_for_each`, we have developed an algorithm that enables iteration over elements in an `std::tuple` and applies an action to them. Key features include:

* Support for both **modifiable** and **constant elements** through **perfect forwarding**. 
* **Automatic skipping of incompatible elements**, with an optional **strict mode** to enforce compatibility. 
* **`constexpr` compatibility**, allowing usage at both compile-time and runtime. 

**Next:** The next article explores `tuple_find`, a constexpr-capable function for searching values in `std::tuple`, which presents additional challenges and solutions.

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
