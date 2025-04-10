---
title: "Staging Strategy (CTSS): constexpr Conversion of 'int to string_view'"
categories: [C++, Compile-time programming]
tags: [c++, c++20, c++23, constexpr, consteval, compile-time, int, string-view, array, literal-type, lambda, conversion, staging, strategy, nttp, ctss, constant-expression]
description: "A detailed introduction to the Compile-Time Staging Strategy (CTSS) for converting an int to a std::string_view at compile time using C++20 and C++23."
image: /assets/images/roadblock.png
---

## Introduction

With the latest `constexpr` extensions in C++, we can perform more and more computations at compile time. However, in practice, we repeatedly encounter a specific problem: What do we do when a non-`constexpr` variable suddenly needs to be `constexpr` later in the program?

**A typical example:** The `size` member of a `std::string` is used to instantiate a `std::array` of the corresponding size. The compiler rejects this because the **size is not a constant expression**.

This is where the **Compile-Time Staging Strategy (CTSS)** comes into play. It allows us to make values `constexpr` in multiple stages.

In this article, we will take a detailed look at how CTSS works, using the example of converting an `int` to a `std::string_view` at compile time.

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
  auto const right_size = rng::distance(rng::cbegin(oversized_buffer),
                                        result.ptr);
  std::array<char, right_size + 1> rightsize_buffer{}; // Error
  ...
}

constexpr auto str_view = int_to_string_view<32, [] {
  return calculation(42);
}>();
```

> However, **this step fails** because the determined size (in our case, the variable `right_size`) **is not a constant expression**.
  {: .prompt-info }

The **Compile-Time Staging Strategy (CTSS)** provides an elegant solution to this problem. It allows us to transform intermediate results   into `constexpr` values in multiple stages, ultimately enabling the creation of a valid `std::string_view`.

## The Compile-Time Staging Strategy (CTSS)

**The first step is done:** We have identified the problem. In our example, the `rightsize_buffer` array expects `right_size` to be a constant expression – but it isn't.

To resolve this issue, we encapsulate the entire code up to the error inside a `constexpr` lambda and return all values that need to be `constexpr`. In our case, this is `right_size`. Additionally, we need to return the oversized array `oversized_buffer` so we can later trim it to the exact required size.

```c++
namespace rng = std::ranges;

template <auto buffer_size, auto int_builder>
consteval auto int_to_string_view() {
  constexpr auto intermediate_result = [] { 
    std::array<char, buffer_size> oversized_buffer{};
    auto const result = std::to_chars(rng::begin(oversized_buffer),
                                      rng::end(oversized_buffer),
                                      int_builder());
    auto const right_size = rng::distance(rng::cbegin(oversized_buffer),
                                          result.ptr);
    return std::pair{oversized_buffer, right_size};
  }();
  ...
}
```

**Important Considerations:**

* All return values **must** be of `literal types`[^1], as only these can be used to initialize `constexpr` variables (such as `intermediate_result`).

* The lambda **must not capture any non-`constexpr` values** from its surrounding scope, as that would make it non-evaluatable at compile time. Although this is not an issue in our example, it is a common source of errors in compile-time programming and should always be kept in mind.

**With this, the first staging step is complete:** We now have the relevant values in a `constexpr`-compatible form and can proceed to the next step — adjusting the array size.

## Adjusting the Array Size

Creating an array with the exact required size is no longer an issue since we now have the precise size available as a `constexpr` value. We reserve an extra byte for the 'null terminator' and copy all characters from the oversized array into the appropriately sized `rightsize_buffer` array.

```c++
namespace rng = std::ranges;

template <auto value> consteval auto& to_static() { return value; }

template <auto buffer_size, auto int_builder>
consteval auto int_to_string_view() {
  constexpr auto intermediate_result = [] { 
    std::array<char, buffer_size> oversized_buffer{};
    auto const result = std::to_chars(rng::begin(oversized_buffer),
                                      rng::end(oversized_buffer),
                                      int_builder());
    auto const right_size = rng::distance(rng::cbegin(oversized_buffer),
                                          result.ptr);
    return std::pair{oversized_buffer, right_size};
  }();

  std::array<char, intermediate_result.second + 1> rightsize_buffer{};
  rng::copy_n(rng::cbegin(intermediate_result.first),
                          intermediate_result.second,
                          rng::begin(rightsize_buffer));
  rightsize_buffer[intermediate_result.second] = '\0';
  return std::string_view{to_static<rightsize_buffer>()}; // Error
}
```
> **At this point, we face another problem:** How do we return a `std::string_view` pointing to an array when that array is a local variable? Simply **declaring the variable as `static` is not allowed** in a `constexpr` context.
{: .prompt-info }

**Two Possible Solutions:**

1. Declare the `rightsize_buffer` array as `static constexpr`[^4].

2. Declare `rightsize_buffer` as `constexpr` and **make the array indirectly `static`** by passing it to the helper function `to_static`. By passing the `constexpr` array as a **Non-Type Template Parameter (NTTP)**, the array is placed in `static` storage, and `to_static` simply returns a reference to this memory.

In both cases, the `rightsize_buffer` array must be declared as `constexpr`. However, directly declaring it as `constexpr` is not possible, as the subsequent copy operation would otherwise fail.

Once again, we face the challenge of a non-`constexpr` variable that needs to become `constexpr` later in the program.

## Final Staging

At this point, we apply the **Compile-Time Staging Strategy** one last time. After identifying the error, we again encapsulate the relevant code in a `constexpr` lambda and return all values that need to be `constexpr`. In this case, it is only the `rightsize_buffer` array itself.

Finally, we pass the array as an NTTP to our helper function `to_static`. The returned reference to the statically stored array is then used to create and return a `std::string_view` instance.

```c++
namespace rng = std::ranges;

template <auto value> consteval auto& to_static() { return value; }

template <auto buffer_size, auto int_builder>
consteval auto int_to_string_view() {
  constexpr auto intermediate_result = [] { 
    std::array<char, buffer_size> oversized_buffer{};
    auto const result = std::to_chars(rng::begin(oversized_buffer),
                                      rng::end(oversized_buffer),
                                      int_builder());
    auto const right_size = rng::distance(rng::cbegin(oversized_buffer),
                                          result.ptr);
    return std::pair{oversized_buffer, right_size};
  }();

  constexpr auto rightsize_buffer = [&intermediate_result] {
    std::array<char, intermediate_result.second + 1> rightsize_buffer{};
    rng::copy_n(rng::cbegin(intermediate_result.first),
                intermediate_result.second,
                rng::begin(rightsize_buffer));
    rightsize_buffer[intermediate_result.second] = '\0';
    return rightsize_buffer;
  }();

  return std::string_view{to_static<rightsize_buffer>()};
}

auto main() -> int {
  constexpr auto str_view = int_to_string_view<32, [] {
    return calculation(42);
  }>();
  ...
}
```

Unlike the first staging step, this time **we capture an external variable** inside the lambda. However, this is completely safe because it only captures the `constexpr` variable `intermediate_result`. This ensures that the lambda remains `constexpr`-evaluatable, avoiding the previously mentioned pitfall.

With this last step, the conversion is complete. We have successfully transformed an **`int`** into a `constexpr`-evaluatable **`std::string_view`** while ensuring that all required values are truly `constexpr`.

## Summary of the Compile-Time Staging Strategy (CTSS)

1. **Identify the error** – Analyze which variable is not `constexpr` but needs to be.

2. **Encapsulate code** – Extract the affected code into a `constexpr` lambda that returns the necessary values.

3. **Watch for pitfalls** – Ensure that no non-`constexpr` values are captured and that only `literal types` are returned.

4. **Store values in `constexpr` variables** – Use the returned values to initialize `constexpr` variables.

5. **Final processing** – Use the now `constexpr`-compatible values for the actual computation, such as creating a `std::string_view`.

## Conclusion

The **Compile-Time Staging Strategy** is a useful technique for many scenarios where `constexpr` constraints in C++ seem to pose a challenge. It allows us to solve complex problems and unlocks new possibilities for optimized, efficient programs. With the continuous improvements in `C++20` and `C++23`, compile-time programming is becoming increasingly powerful—and strategies like **CTSS help us** make the most of it.

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

[^4]: **Since C++23, it is allowed to declare variables as `static constexpr` in a `constexpr` context**. However, we do not use this approach because the Clang compiler currently has issues handling `static constexpr` inside `consteval` functions. 
