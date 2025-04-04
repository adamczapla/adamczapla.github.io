---
title: "NTTP Builder Strategy – constexpr Conversion of 'vector to array'"
categories: [C++, Compile-time programming]
tags: [c++, c++20, constexpr, compile-time, builder, array, vector, literal-type, lambda, conversion, staging, strategy, nttp, constant-expression]
description: "A deep dive into the 'NTTP Compile-Time Builder Strategy' for constexpr conversion of 'std::vector' to 'std::array' in C++."
image: /assets/images/intermediate-step.png
---

## Introduction

Non-Type Template Parameters (NTTPs) are a powerful feature of compile-time programming in C++. They allow values to be passed and processed at compile time. However, they come with a significant limitation: **only** literal types[^1] are allowed.

This means that many commonly used standard types, such as `std::string` or `std::vector`, **cannot be passed as NTTPs**. In many use cases, it would be beneficial to utilize dynamic or complex data types in a similar manner.

The **NTTP Compile-Time Builder Strategy** provides a way to achieve exactly that.

This article explains how this technique works and in which scenarios it can be effectively used.

## Core Concept: The NTTP Compile-Time Builder Strategy

The NTTP Compile-Time Builder Strategy consists of the following steps:

**1. Encapsulation of Value Creation in a Builder**

Instead of passing the value directly, we define a `constexpr` lambda function that generates and returns the desired value. This lambda acts as a **builder**, describing **how the value is created** without instantiating it immediately.

```c++
constexpr auto str_builder = [] { return std::string{"Hello NTTP!"}; };
```

**2. Passing the Builder as an NTTP**

Since lambdas in C++ are implicitly `constexpr`, they can be passed as NTTPs. This allows non-literal types to be processed at compile time indirectly.

```c++
constexpr auto str_builder = [] { return std::string{"Hello NTTP!"}; };

auto main() -> int {
  process_string<str_builder>(); // Passing the Builder as an NTTP
}
```

**3. Generating the Value within the Function**

Inside the target function (`process_string`), the builder is executed to generate the desired value at compile time.

```c++
template <auto builder>
auto process_string() {
  std::string str = builder(); // Generate the value using the builder
  // Further processing...
}
```

After explaining the fundamentals of the NTTP Compile-Time Builder Strategy, let’s look at a concrete use case.

## Practical Example: Converting `std::vector<int>` to `std::array<int, N>`

### Implementation:

```c++
template <size_t max_size, auto builder>
constexpr auto to_array() noexcept {
  namespace rng = std::ranges;
  constexpr auto data = [] {
    auto const int_vec = builder();
    std::array<int, max_size> result{};
    auto const end_pos = rng::copy(int_vec, rng::begin(result)).out;
    auto const right_size = rng::distance(rng::begin(result), end_pos);
    return std::pair{result, right_size};
  }();
  std::array<int, data.second> result{};
  rng::copy_n(rng::begin(data.first), data.second, rng::begin(result));
  return result;
}
```

### How Does the Conversion Work?

The `to_array` function applies the **NTTP Compile-Time Builder Strategy** to transform a `std::vector<int>` into a `std::array<int, N>` at compile time.

The first NTTP defines the size of the array. This size is not automatically derived but must be explicitly passed at the function call. It must be large enough to hold the entire contents of the `std::vector`. The size cannot be inferred from `std::vector` because `std::array` requires a constant expression for its size.

The second NTTP is the **builder**, which describes how the `std::vector` is created.

Inside `to_array`, the builder executes to generate a `std::vector` at compile time.

The contents of the vector are copied into an **oversized array** (`std::array<int, max_size>`), and the exact number of copied elements is determined.

These values — the temporary array and its corresponding element count — are returned in a `std::pair`.

### Why This Intermediate Step?

The key limitation of `std::array` is that its **size must be known at compile time**. We can only create the final array once its exact size is available as a constant expression. To achieve this, the entire process is **wrapped** in a lambda function, a technique known as **Compile-Time Staging Strategy (CTSS)**.

> **For more details on CTSS:**
> 
> [Compile-Time Staging Strategy (CTSS): constexpr Conversion of int to std::string_view](https://adamczapla.github.io/posts/compile-time-staging-strategy-ctss-constexpr-conversion-of-int-to-string-view/)
  {: .prompt-info }

In the final step, the temporary oversized array is **trimmed** to its actual size and returned as a `constexpr` value.

### Usage Example:

```c++
constexpr auto vector_builder = [] {
  std::vector<int> vec{0, 8, 15};
  vec.push_back(50);
  // Familiar handling of std::vector
  return vec;
};

static constexpr auto arr = to_array<42, vector_builder>();
```

Now, the builder’s result is available as a `constexpr` value and can be further processed.

### When is such a conversion useful?

Since C++20, many types that were previously restricted to runtime, including `std::vector`, can now be used in `constexpr` contexts — even though they involve dynamic memory management. This makes it possible to work with `std::vector` at compile-time just as flexibly as at runtime.

The advantages are clear:

* **Dynamic memory management** → Elements can be added or removed dynamically.
* **Flexibility** → The number of elements does not need to be known in advance.

However, there is a critical limitation:

Dynamically allocated memory **must** also be deallocated at compile-time. This means that `std::vector` cannot retain its values beyond compile-time, as its allocated memory is freed once the `constexpr` execution is complete.

To store values permanently in a compile-time data structure, we need to convert `std::vector` into an `std::array`.

## When Is the NTTP Compile-Time Builder Strategy Necessary?

The NTTP Compile-Time Builder Strategy is a powerful tool, but it is not always required. In some cases, a `std::vector` can simply be passed as a regular function argument without needing a builder.

However, in this specific example, passing a `std::vector` as a function argument would not be possible because the lambda inside `to_array` would need to capture it by reference. Since function parameters are never `constexpr`, capturing a reference to the `std::vector` would create a non-`constexpr` reference, which is not allowed inside a `constexpr` function.

This approach is particularly beneficial when combined with the [Compile-Time Staging Strategy (CTSS)](https://adamczapla.github.io/posts/compile-time-staging-strategy-ctss-constexpr-conversion-of-int-to-string-view/). Whenever the generated value needs to be used in a **context that initializes a `constexpr` variable**, the NTTP Compile-Time Builder Strategy fully demonstrates its advantages.

## Conclusion

This article has demonstrated how the NTTP Compile-Time Builder Strategy enables the use of non-literal types as NTTPs, allowing dynamic structures like `std::vector` to be utilized in pure compile-time processing.

In combination with [Compile-Time Staging Strategy (CTSS)](https://adamczapla.github.io/posts/compile-time-staging-strategy-ctss-constexpr-conversion-of-int-to-string-view/), this technique provides a flexible and efficient way to manage complex data transformations at compile time, ensuring that dynamically generated values remain valid beyond their immediate execution scope.

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

## Footnote

{% include footnote-literal-types.md %}

