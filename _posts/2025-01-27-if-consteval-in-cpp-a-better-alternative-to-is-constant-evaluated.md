---
title: "'if consteval' in C++: A Better Alternative to is_constant_evaluated"
date: 27-01-2025 12:00:00 +0100
categories: [C++, Compile-time programming]
tags: [c++, c++23, if-consteval, is-constant-evaluated, constexpr, consteval, compile-time, static-analysis, consteval-functions]
description: "A detailed exploration of 'if consteval' and its advantages over std::is_constant_evaluated() in C++."
image: /assets/images/consteval.png
---

## Introduction

When working with `constexpr` and `consteval` in C++, developers may run into some limitations when attempting to evaluate conditions at compile-time. One common scenario is using `std::is_constant_evaluated()` to differentiate between compile-time and runtime code. However, there are cases where this approach fails due to how the compiler performs **static analysis** — leading to behavior that might be surprising at first glance.

Let’s take a closer look at what can go wrong and how `if consteval` can help.

## Why Doesn't This Code Work?

Consider the following `constexpr` function:

```c++
consteval auto consteval_func(int val) { return val; }

constexpr auto constexpr_func(int val) {
  if (std::is_constant_evaluated()) {
    return consteval_func(val);
  } else {
    std::cout << "is_constant_evaluated==false\n";
    return 0;
  }
}
```

The compiler produces an error on the line (**`5`**), because the parameter `val` is not `constexpr`. A `consteval` function like `consteval_func()` can only be called with `constexpr` arguments.

Logically, this seems unnecessary since the `if (std::is_constant_evaluated())` branch will only execute at compile-time. And even though `val` isn’t `constexpr`, the `constexpr` function would have been invoked with a `constexpr` argument if executed at compile-time. For example:

```c++
constexpr auto val = [] consteval { return constexpr_func(42); }();
```

## Why doesn't it work anyway?

The compiler reports an error during **static analysis**, even if the function is not invoked. The error arises because the compiler, during its static analysis phase, checks the code for syntactic and semantic correctness. Since an `if` clause is a runtime construct, the compiler does not generate separate code branches for compile-time and runtime during this phase.

Instead, the compiler treats the call `consteval_func(val)` in the `if` clause as if it were outside the clause entirely. This means the call is always analyzed, regardless of whether it will actually execute or not. Consequently, the compiler raises an error because `val` is not `constexpr`.

> This behavior leads to the conclusion that calling a `consteval` function from a `constexpr` function with a non-`constexpr` argument is **always** invalid.
{: .prompt-info }

## Enter `if consteval`

Here’s how `if consteval` resolves the issue:

```c++
consteval auto consteval_func(int val) { return val; }

constexpr auto constexpr_func(int val) {
  if consteval {
    return consteval_func(val);
  } else {
    std::cout << "is_constant_evaluated==false\n";
    return 0;
  }
}
```

With `if consteval`, the compiler makes the decision about which branch to execute during **static analysis**. This means:

* The `consteval` branch is **excluded** entirely from runtime evaluation.
* The compiler knows that the call to `consteval_func(val)` happens only in a compile-time context, allowing it to validate the code confidently.

Unlike an `if` clause, `if consteval` is a compile-time construct, just like `if constexpr`. The compiler decides which branch to execute during **static analysis** and **excludes** the other entirely. This means the compiler can safely allow the call to the `consteval_func(val)` function, knowing that the call will occur only in a `constexpr` context during compile-time.

## Conclusion

`std::is_constant_evaluated()` is useful for distinguishing between compile-time and runtime logic, but it is not suitable when working with `consteval` functions. The introduction of `if consteval` in C++ provides a cleaner and safer way to handle compile-time evaluation, ensuring that the compiler can confidently differentiate between branches.

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
