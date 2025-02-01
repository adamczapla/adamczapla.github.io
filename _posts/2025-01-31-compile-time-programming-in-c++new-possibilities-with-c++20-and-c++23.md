---
title: "Compile-Time Programming in C++: New Possibilities with C++20 and C++23"
date: 31-01-2025 12:00:00 +0100
categories: [C++, Compile-time programming]
tags: [c++, c++20, c++23, constexpr, consteval, compile-time, prvalue, lambda, string, string-view, array]
description: "A detailed exploration of ..."
---

## Introduction

Efficiency is a **key factor** in modern C++ projects. Especially in performance-critical applications, it is beneficial to avoid expensive memory allocations and runtime computations. The new features in `C++20` and `C++23` greatly expand compile-time programming, allowing even non-literal types like `std::string` and `std::vector` to be processed at compile time.

In this article, I will show a concrete example of how to efficiently convert `std::string` into `std::string_view` at compile time. This reduces runtime costs, avoids unnecessary dynamic memory allocations, and enables new optimizations – such as for logging or generated code metadata. In addition to the new language features, I will explain fundamental concepts of compile-time programming and present practical solutions to common challenges.

## Challenge: Using `std::string` as `constexpr`

Every call to a `constexpr` or `consteval` function from a non-`constexpr` context requires all function arguments to be `literal types`[^1], meaning their values must be known at compile time.

However, `std::string` is not a literal type, even though it has `constexpr` constructors since `C++20` and can be used in a `constexpr` context at compile time.

Let's consider the following example:

```c++
auto main() -> int { // non-constexpr context
  std::string str{"hello world"}; 
  constexpr auto str_view = to_string_view(str);
  return 0;
}
```

Here, the call fails because `std::string` cannot be used as an argument in a `constexpr` function to initialize a `constexpr` variable.

It also does not help to declare the variable `str` as `constexpr`, as shown in the following example:

```c++
auto main() -> int { // non-constexpr context
  constexpr std::string str{"hello world"};
  constexpr auto str_view = to_string_view(str);
  return 0;
}
```

### 1. Passing a `prvalue` instead of an `lvalue`

Instead of passing a variable, we use a temporary value (`prvalue`):

```c++
auto main() -> int { // non-constexpr context
  constexpr auto str_view = to_string_view(std::string{"hello world"});
  return 0;
}
```

Since `C++17`, `prvalues` are no longer objects but pure expressions. The materialization into an object occurs only within the `constexpr` function `to_string_view`, making the code valid because the temporary `std::string` does not need to be `constexpr` at the time of the call.

### 2. Using a Lambda Function

Alternatively, the `std::string` can be encapsulated in a lambda:

```c++
auto main() -> int { // non-constexpr context
  constexpr auto str_view = [] {
    std::string str{"hello world"};
    return to_string_view(str);
  }();
  return 0;
}
```

Since lambdas have been implicitly constexpr since `C++17`, the call to the `constexpr` function takes place from a `constexpr` context.

## Converting `std::string` into `std::array`

To implement `to_string_view`, we first convert `std::string` into `std::array`. 

```c++
namespace rng = std::ranges;

template <auto max_size> 
consteval auto to_string_view(std::string const& str) {
  std::array<char, max_size> max_size_array{};
  rng::copy(str, max_size_array.begin());
  return max_size_array;
}

auto main() -> int { // non-constexpr context
  constexpr auto str_view = to_string_view<128>(std::string{"hello world!"});
  return 0;
}
```

> The key limitation of `std::string` and other non-literal types is that they **must** deallocate their memory in a `constexpr` context. If their values need to leave the `constexpr` context, they **must** be copied into a literal type.
{: .prompt-info }

`std::array` is therefore an ideal choice for storing the `std::string` value. The maximum size of the array is passed as a **Non-Type Template Parameter (NTTP)** because function parameters in C++ can **never** be `constexpr` and when instantiating the `right_size_array` array, `max_size` must be a constant expression.

### Dynamic Adjustment of Array Size

Since we never know the exact array size in advance, we first create an oversized array and then trim it to the exact size.

```c++
namespace rng = std::ranges;

template <auto max_size> 
constexpr auto to_oversized_array(std::string const& str) {
  std::array<char, max_size> max_size_array{};
  auto const end_pos = rng::copy(str, rng::begin(max_size_array));
  auto const right_size = rng::distance(rng::cbegin(max_size_array), end_pos.out);
  return std::pair{max_size_array, right_size};
}

template <auto max_size> 
consteval auto to_string_view(std::string const& str) {
  constexpr auto intermediate_data = to_oversized_array<max_size>(str);
  std::array<char, intermediate_data.second> right_size_array{};
  rng::copy_n(rng::cbegin(intermediate_data.first), intermediate_data.second,
              rng::begin(right_size_array));
  return right_size_array;
}
```

### Problem: Function Parameters in C++ Are Never `constexpr`

The function parameter `str` is not `constexpr`, yet we pass it as an argument to a `constexpr` function that initializes a `constexpr` variable. However, the variable `intermediate_data` **must** remain `constexpr` because, when instantiating the `right_size_array` array (`line 14`), the size **must** be a constant expression.

### Solution: Lambda as NTTP

Instead of passing `std::string` as a parameter, we encapsulate it in a lambda and pass it as an NTTP.

```c++
namespace rng = std::ranges;

template <auto max_size, auto string_builder> 
constexpr auto to_oversized_array() {
  std::array<char, max_size> max_size_array{};
  auto const end_pos = rng::copy(string_builder(), rng::begin(max_size_array));
  auto const right_size = rng::distance(rng::cbegin(max_size_array), end_pos.out);
  return std::pair{max_size_array, right_size};
}

template <auto max_size, auto string_builder> 
consteval auto to_string_view() {
  constexpr auto intermediate_data = to_oversized_array<max_size, string_builder>();
  std::array<char, intermediate_data.second> right_size_array{};
  rng::copy_n(rng::cbegin(intermediate_data.first), intermediate_data.second,
              rng::begin(right_size_array));
  return right_size_array;
}

auto main() -> int { // non-constexpr context
  constexpr auto str_view = to_string_view<128, [] { return std::string{"hello world!"}; }>();
  return 0;
}
```

### Optimization: Optional

To keep everything in place, we encapsulate the function `to_oversized_array` in a lambda.

```c++
namespace rng = std::ranges;

template <auto max_size, auto string_builder> 
consteval auto to_string_view() {
  constexpr auto intermediate_data = [] {
    std::array<char, max_size> max_size_array{};
    auto const end_pos = rng::copy(string_builder(), rng::begin(max_size_array));
    auto const right_size = rng::distance(rng::cbegin(max_size_array), end_pos.out);
    return std::pair{max_size_array, right_size};
  }();
  std::array<char, intermediate_data.second> right_size_array{};
  rng::copy_n(rng::cbegin(intermediate_data.first), intermediate_data.second,
  rng::begin(right_size_array));
  return right_size_array;
}
```

## Converting `std::array` into `std::string_view`

By marking the array `right_size_array` with `static constexpr`, we store it in **static memory** and allow it to be referenced using a `std::string_view` instance. This instance is then returned to the caller of the `to_string_view` function.

```c++
namespace rng = std::ranges;

template <auto max_size, auto string_builder> 
consteval auto to_string_view() {
  constexpr auto intermediate_data = [] {
    std::array<char, max_size> max_size_array{};
    auto const end_pos = rng::copy(string_builder(), rng::begin(max_size_array));
    auto const right_size = rng::distance(rng::cbegin(max_size_array), end_pos.out);
    return std::pair{max_size_array, right_size};
  }();

  static constexpr auto right_size_array = [&intermediate_data] {
    std::array<char, intermediate_data.second> right_size_array{};
    rng::copy_n(rng::cbegin(intermediate_data.first), intermediate_data.second,
                rng::begin(right_size_array));
    return right_size_array;
  }();

  return std::string_view{right_size_array}; 
}
```

### Portability: Using `to_static` for Clang

Since Clang has issues with `static constexpr` in `consteval` functions in C++23, the following helper function ensures portability:

```c++
template <auto value> consteval auto& to_static() { return value; }
```

We call this function with the array `right_size_array` as a **Non-Type Template Parameter (NTTP)**. NTTPs allow values to be stored directly in the static memory area, making them referenceable. This way, we can safely store `std::array` data in static memory and return it as `std::string_view`.

```c++
namespace rng = std::ranges;

template <auto value> consteval auto& to_static() { return value; }

template <auto max_size, auto string_builder>
consteval auto to_string_view() {
  constexpr auto intermediate_data = [] {
    std::array<char, max_size> max_size_array{};
    auto const end_pos = rng::copy(string_builder(), rng::begin(max_size_array));
    auto const right_size = rng::distance(rng::cbegin(max_size_array), end_pos.out);
    return std::pair{max_size_array, right_size};
  }();

  constexpr auto right_size_array = [&intermediate_data] {
    std::array<char, intermediate_data.second> right_size_array{};
    rng::copy_n(rng::cbegin(intermediate_data.first), intermediate_data.second,
                            rng::begin(right_size_array));
    return right_size_array; 
  }();

  return std::string_view{to_static<right_size_array>()};
}
```

With this, our compile-time conversion from `std::string` to `std::string_view` is not only complete but also portable and efficient.

## Use Case: Compile-Time Generation of Log Tags

A practical example is creating log tags for generic types:

```c++
template <typename T>
constexpr auto type_name() { // GCC only
  constexpr std::string_view prefix = "constexpr auto type_name() [with T = ";
  std::string_view name = __PRETTY_FUNCTION__;
  name.remove_prefix(prefix.size());
  name.remove_suffix(1);
  return name;
}

template <typename T>
constexpr auto log_tag() {
  return to_string_view<64, [] { return "Log<" + std::string(type_name<T>()) + ">"; }>();
}

auto main() -> int { // non-constexpr context
  static constexpr auto log_string = log_tag<std::vector<std::string>>();
  // output: Log<std::vector<std::__cxx11::basic_string<char> >>
  return 0;
}
```

Here, a *log tag* for a generic type is created at compile time. This reduces runtime costs and avoids unnecessary memory allocations.

## Conclusion

The new features in `C++20/23` enable powerful compile-time manipulations even for non-literal types. The techniques shown allow efficient conversion of `std::string` into `std::string_view`, reducing runtime costs.

Especially in performance-critical applications, compile-time programming can provide significant advantages. The ability to process strings efficiently at compile time opens up exciting optimization possibilities – not only for logging but also for many other areas of modern C++ development.

## Footnote

[^1]: **What are Literal Types?**

    A literal type in C++ is a type that can be used in a constexpr context, meaning inside constant expressions. This includes:
    * Built-in types such as int, char, double, bool, and nullptr_t
    * Enumerations (enum and enum class)
    * Pointer types to literal types, including const and nullptr_t pointers
    * Pointers to members of literal types
    * Literal classes. (For more details on `literal classes`, see the section below.)



> ### Requirements for a class to be a literal class

**`1.`** All non-static members must be literals.
>
`2.` The class must have at least one user-defined constexpr constructor, or all non-static members must be initialized in-class.
>
`3.` In-class initializations for non-static members of built-in types must be constant expressions.
>
4. In-class initializations for non-static members of class types must either use a user-defined constexpr constructor or have no     initializer. If no initializer is given, the default constructor of the class must be constexpr. Alternatively, all non-static members of the class must be initialized in-class.
{: .prompt-info }

5. A constexpr constructor must initialize every member or at least those that are not initialized in-class.

6. Virtual or normal default destructors are allowed, but user-defined destructors with {} are not allowed. User-defined constructors with {} are allowed if they are declared as constexpr. However, user-defined constexpr destructors in literal classes are often of limited use because literal classes do not manage dynamic resources. In non-literal classes, however, they can be important, especially for properly deallocating dynamic resources in a constexpr context.

7. Virtual functions are allowed, but pure virtual functions are not.

8. Private and protected member functions are allowed.

9. Private and protected inheritance are allowed, but virtual inheritance is not.

10. Aggregate classes with only literal non-static members are also considered literal classes. This applies to all aggregate classes without a base class or if the base class is a literal class.

11. Static member variables and functions are allowed if they are constexpr and of a literal type.

12. Friend functions are allowed inside literal classes.

13. Default arguments for constructors or functions must be constant expressions.
    
