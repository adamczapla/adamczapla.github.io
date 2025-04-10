---
title: "A Constexpr-Compatible Algorithm for std::tuple - Part 2: tuple_find"
categories: [Compile-time programming, Algorithms for Heterogeneous Containers]
tags: [c++, constexpr, compile-time, lambda, tuple, std-tuple, algorithms, heterogeneous-containers, tuple-for-each, tuple-find, std-apply, fold-expressions, parameter-pack, variadic-template, comma-operator, value-category, static_assert, reference-wrapper, std-variant, std-optional, index-sequence]
description: "A detailed guide to implementing the constexpr-compatible search algorithm 'tuple_find' for heterogeneous containers using modern C++."
image: /assets/images/tuple_find.png
---

## Introduction

In the [first part](https://adamczapla.github.io/posts/implementing-constexpr-compatible-algorithms-for-heterogeneous-containers-tuple-for-each/) of this article series, an algorithm called `tuple_for_each` was introduced, enabling iteration over all elements of a `std::tuple` and applying an action to each one - without relying on classic loops or iterators. Based on that foundation, this second article covers another common use case for heterogeneous containers: **searching for a specific value**.

In standard containers like `std::vector`, this task is typically solved using `std::find` or `std::ranges::find`. For `std::tuple`, however, no such equivalent exists. This article **closes that gap** by explaining step by step how such a function can be implemented.

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

## Reference Types and Return Value

Since `tuple_find` returns a reference to the matched element, it's important to distinguish whether this reference is `const` or **modifiable**. However, this information is only available **at the time a match is found**.

The search itself takes place - just like in `tuple_for_each` - **inside a fold expression using the comma operator**. Such an expansion **cannot be exited early**. Therefore, any potential match must be **stored immediately in a suitable data structure**, even though the fold expression continues afterward.

This requirement makes it necessary to **fully declare the return type before the search begins**. Since the reference type (`const` or not) is still unknown at this point, we use `std::variant` to store either a `const` or non-`const` reference. To additionally signal whether a match was found, `std::variant` is combined with `std::optional`:

```c++
std::optional<std::variant<non_const_result, const_result>>
```

The reference is stored using `std::reference_wrapper`, because a `std::pair` does not allow assignment when one of its elements is of type `const&`. **Such an assignment is required inside the fold expression.** `std::reference_wrapper` circumvents this limitation and makes the assignment possible.

## Fold Expression: Type Check and Comparison

At the beginning of the fold expression, a **type check** is performed:

```c++
([&] {
  if constexpr (std::is_same_v<std::remove_cvref_t<decltype(tuple_values)>, value_t>) {
    // Comparison goes here
  }
}(), ...);
```

Only elements whose type **exactly matches** the type of the search value are included in the comparison. This restriction is a direct result of how the function is structured: the return type must be defined **before** the iteration over the tuple begins. Since it is declared based on `value_t`, this is the only type that can be used for a valid reference return. Therefore, the comparison is explicitly limited to elements of this exact type.

Since the function returns a **reference** to the found element, an exact type match is required—otherwise, no valid binding would be possible. For example, an `int` element cannot be returned as a `long&`.

Because a fold expression **cannot be exited early**, any match must be stored **immediately** in a previously defined data structure. The comparison is therefore performed **only on elements where the types are guaranteed to match**.

Thanks to the previously constructed **mapping between tuple elements and their positions** (via `std::index_sequence`), each element also has a corresponding index. This allows the search to be started from a specific position within the tuple.

```c++
if (!result && idx >= index && std::equal_to{}(value, tuple_values)) {
  ...
}
```

This checks whether an element matches the search value **and** whether it appears at or **after** the specified start index. Only if both conditions are met and no previous match has been recorded, the result is stored.

Since the matched element might be either `const` or **modifiable**, the storage is handled accordingly:

```c++
if constexpr (std::is_const_v<std::remove_reference_t<decltype(tuple_values)>>) {
  result = result_t{std::in_place, const_result_t{tuple_values, idx}};
} else {
  result = result_t{std::in_place, non_const_result_t{tuple_values, idx}};
}
```

This ensures that the return value reflects **both** the correct reference type (`const` or not) **and** the element’s position within the tuple.

## Complete Function

```c++
template <typename tuple_t, std::equality_comparable value_t>
constexpr auto tuple_find(tuple_t&& tuple, value_t const& value, size_t index = 0) noexcept {
  return [&]<size_t... idx>(std::index_sequence<idx...>) {
    return std::apply([&](auto&&... tuple_values) {
      static_assert(((std::is_lvalue_reference_v<decltype(tuple_values)>) && ...) ,
        "Error: All tuple elements must be lvalue references to ensure that returned "
        "references remain valid.");

      using non_const_result_t = std::pair<std::reference_wrapper<value_t>, size_t>;
      using const_result_t = std::pair<std::reference_wrapper<std::add_const_t<value_t>>, size_t>;
      using result_t = std::optional<std::variant<non_const_result_t, const_result_t>>;
            
      result_t result{};

      ([&] {
        if constexpr (std::is_same_v<std::remove_cvref_t<decltype(tuple_values)>, value_t>) {
          if (!result && idx >= index && std::equal_to{}(value, tuple_values)) {
            if constexpr (std::is_const_v<std::remove_reference_t<decltype(tuple_values)>>) {
              result = result_t{std::in_place, const_result_t{tuple_values, idx}};
            } else {
              result = result_t{std::in_place, non_const_result_t{tuple_values, idx}};
            }
          }
        }
      }() , ...);

      return result;
    }, std::forward<tuple_t>(tuple));
  }(std::make_index_sequence<std::tuple_size<std::remove_cvref_t<tuple_t>>{}>{});
}
```

## Example Use Cases

Now that the `tuple_find` function is fully implemented and all important design decisions have been discussed, the following examples illustrate its practical use. The examples cover both typical **runtime scenarios** and `constexpr` **use cases**.

### 1. Search and Modify Values (Runtime)

This example shows how to find multiple occurrences of a value in a `std::tuple` and modify them afterward. The return value is a **real reference** - any changes affect the underlying variables directly:

```c++
int a{20}, b{10}, c{10}, d{30};
auto tpl = std::tie(a, b, c, d);

size_t idx = 0;
while (true) {
  auto result = tuple_find(tpl, 10, idx);
  if (!result) break;
  auto& ref = std::get<0>(*result); // Access non-const reference
  std::cout << "Found: " << ref.first << '\n';
  ref.first += 100; // Modify
  idx = ref.second + 1; // Continue search from next index
}
std::cout << "New values: b = " << b << ", c = " << c << '\n';
```

**Output:**

```
Found: 10  
Found: 10  
New values: b = 110, c = 110
```

### 2. `constexpr` Support

Since `tuple_find` is fully `constexpr` - **compatible**, the search can also be executed at compile time. For example:

```c++
static constexpr int x{10}, y{20}, z{30};
constexpr auto tpl = std::tie(x, y, z);
constexpr auto result = tuple_find(tpl, 10);

if constexpr (result) {
  static_assert(std::get<1>(*result).second == 0); // Match at position 0
}
```

This allows the function to be used inside `static_assert` or other `consteval` contexts.

### 3. Reference Type Depends on Search Match

The return type of `tuple_find` depends on the **first matching element**: If it is `const`, a `const` reference is returned—otherwise a modifiable one.

```c++
int modifiable{10};
const int immutable{10};
auto tpl = std::tie(immutable, modifiable);

auto result = tuple_find(tpl, 10);
if (result) {
  if (auto ptr = std::get_if<0>(&*result)) {
    std::cout << "non-const: " << ptr->first << '\n';
  } else {
    std::cout << "const: " << std::get<1>(*result).first << '\n';
  }
}
```

**Output:**

```
const: 10
```

## Conclusion

`tuple_find` is specifically designed for **heterogeneous containers** like `std::tuple` and introduces several characteristics that arise from C++'s **type system** and **reference semantics**:

* Only elements whose type exactly matches the search value are considered. A search for `int` will **not** find a `double`, even if an implicit conversion would be allowed.
* The return type is a `std::variant` that contains either a const or **non**-`const` **reference**. To safely access the result, the user needs to check which alternative was returned using `std::get_if` or `std::holds_alternative`.
* A more generic version based on concepts like `std::equality_comparable_with` or `std::convertible_to` is not possible, since the reference type must be **fully defined before the iteration starts**. This is a limitation of using **fold expressions**, which cannot be short-circuited.

Despite these limitations, `tuple_find` offers clear advantages:

* The algorithm is fully `constexpr`-**compatible**
* It allows **direct reference access** to found elements
* It supports **multiple matches** using a start index
  
Together, these features provide a precise and modern solution for working with `std::tuple`, addressing common real-world problems **without sacrificing clarity or performance**.

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
