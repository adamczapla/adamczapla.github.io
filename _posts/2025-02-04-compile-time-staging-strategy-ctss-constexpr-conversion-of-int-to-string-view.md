---
title: "Compile-Time Staging Strategy (CTSS): constexpr Conversion of 'int to std::string_view'"
categories: [C++, Compile-time programming]
tags: [c++, c++20, c++23, constexpr, consteval, compile-time, int, string-view, array, literal-type, lambda, conversion]
description: "A detailed introduction to the Compile-Time Staging Strategy (CTSS) for converting an int to a std::string_view at compile time using C++20 and C++23."
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

1. Declare the `rightsize_buffer` array as `static constexpr`[^2].

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

## Footnote

[^1]: **What are Literal Types?**

    A literal type in C++ is a type that can be used in a `constexpr` context, meaning inside `constant expressions`. This includes:
    * Built-in types such as `int`, `char`, `double`, `bool`, and `nullptr_t`
    * Enumerations (`enum` and `enum class`)
    * `Pointer` types to literal types, including `const` and `nullptr_t` pointers
    * `Pointers to members` of literal types
    * `Literal classes`[^3]

[^2]: **Since C++23, it is allowed to declare variables as `static constexpr` in a `constexpr` context**. 

    However, we do not use this approach because the Clang compiler currently has issues handling `static constexpr` inside `consteval`   functions. 
    

[^3]: **Requirements for a class to be a `literal class`**

    * All `non-static` members must be literals.
    * The class must have at least one user-defined `constexpr` constructor, or all `non-static` members must be initialized `in-class`.
    * `In-class` initializations for `non-static` members of `built-in` types must be `constant expressions`.
    * `In-class` initializations for `non-static` members of class types must either use a user-defined `constexpr` constructor or have no     initializer. If no initializer is given, the default constructor of the class must be `constexpr`. Alternatively, all `non-static` members of the class must be initialized `in-class`.
    * A `constexpr` constructor must initialize every member or at least those that are not initialized `in-class`.
    * Virtual or normal default destructors are allowed, but user-defined destructors with {} are not allowed. User-defined     constructors with {} are allowed if they are declared as `constexpr`. However, user-defined `constexpr` destructors in literal classes are often of limited use because literal classes do not manage dynamic resources. In `non-literal` classes, however, they can be important, especially for properly deallocating dynamic resources in a `constexpr` context.
    * `Virtual` functions are allowed, but `pure virtual` functions are not.
    * `Private` and `protected` member functions are allowed.
    * `Private` and `protected` inheritance are allowed, but `virtual` inheritance is not.
    * `Aggregate classes`[^4] with only literal `non-static` members are also considered literal classes. This applies to all aggregate classes without a base class or if the base class is a literal class.
    * `Static` member variables and functions are allowed if they are `constexpr` and of a literal type.
    * `Friend` functions are allowed inside literal classes.
    * Default arguments for constructors or functions must be `constant expressions`.
    
    > A literal type ensures that objects of this type can be evaluated at compile time, as long as all dependent expressions are `constexpr`. 
    {: .prompt-info }

[^4]: **Requirements for a class to be a `aggregate class`**
    
    1. **What is allowed:**
    * `Public` members
    * `User-declared` destructor
    * `User-declared` copy and move assignment operators
    * Members can be `not literals`
    * `Public` inheritance
     *  `Protected` or `private` **static** members
    
    2. **What is not allowed:**
    * `Protected` and `private` **non-static** members
    * `User-declared` constructors
    * `Virtual` destructor
    * `Virtual` member functions
    * `Protected/private` or `virtual` inheritance
    * Inherited constructors (by `using` declaration)
    
    3. **Restrictions for the base class:**
    * Only `public` **non-static** members allowed OR
    * `Public` constructor required for `non-public` **non-static** members
