---
title: "Path conflict detection in C++ using a trie-based algorithm"
categories: [Filesystem, Tree-Based Algorithms]
tags: [c++, c++17, trie, tree-algorithm, std-filesystem, path, conflict-detection, configuration-validation, unordered_map, lexically_normal, json, has_conflict]
description: "A concise explanation of 'has_conflict', a trie-based C++ algorithm for detecting overlapping file system paths during configuration validation."
image: /assets/images/tuple_find.png
---

## Introduction

I am currently working on a tool that copies or moves specific directories on the file system. These directories are defined in a JSON file, which contains a list of **source and destination paths** to be processed in order.

However, when more than one path is configured, a possible problem can occur: **two paths might overlap**. For example, if `/a/b` and `/a/b/c` are both listed as sources, it is unclear what should happen if one is moved before the other. This can lead to inconsistent behavior or even **data loss**, especially when moving files.

To detect such conflicts reliably, I wrote a function that stores all paths in a simple tree structure and checks whether a new path overlaps with an existing one. In this article, I show how this function works and why it has proven useful in practice.

## The Trie Structure

To efficiently check for overlapping paths, they are **stored in a tree structure** – more precisely, in a **trie**. Each node in the trie represents a single directory name, that is, one **path component**. This allows each path to be built step by step by inserting its components one after another into the tree.

The structure of a node is kept simple: 

```c++
struct path_node {
  std::unordered_map<std::string, path_node> children{};
  bool is_terminal{false};
};
```

* `children` contains all child nodes – that is, the subdirectories of the current node.
* `is_terminal` marks whether a complete path ends at this point.

When a new path is inserted, all components are traversed – for example `a`, `b`, `c` – and the trie is updated accordingly: if a child node already exists, it is followed; otherwise, a new node is created.

The code for that looks like this:

```c++
auto [child, _] = current_node->children.try_emplace(/* path component */);
current_node = &child->second;
```

`try_emplace` inserts the new node only if it does not already exist – otherwise, it simply returns the existing one. In both cases, `current_node` then **points to the correct child node**. This way, the path is built step by step inside the trie.

With this structure in place, it is now possible to reliably detect whether a new path **is part of** an existing path, **contains** an existing path, or is **identical to** one.

## How paths are traversed

Before we get to the examples, I want to briefly explain how `std::filesystem::path` behaves – especially **how a path is split into individual components**.

When a path like `/a/b` is traversed in a loop

```c++
for (auto const& directory : path) { ... }
```

the iterator yields each path component separately – in this case:

```
/  (root)
a
b
```

This is important because the conflict check is based exactly on this structure: **each of these components is handled and stored separately in the trie**. This is the only way to reliably detect whether a path already exists or overlaps with another one
.

In the next section, I will show how these conflicts are detected using concrete path examples.


## Conflict Detection Examples

The following section shows how **conflict detection** works in detail using concrete path examples. Each case represents a typical situation that can occur in a configuration.

Step by step, these examples build up the logic behind the `has_conflict` function – starting from simple cases and gradually covering all necessary checks.

### Example 1: A simple path (`/a/b`)

The path `/a/b` is the first one to be inserted – the trie is still empty. The nodes `a` and then `b` are created step by step, and `b` is marked as terminal at the end.

No conflict can occur at this point. This case shows the base state: paths are added without interfering with anything.

```c++
namespace fs = std::filesystem;

auto has_conflict(path_node& root_node, fs::path const& path) -> bool {
  path_node* current_node = &root_node;

  for (auto const& directory : path) {
    auto [child, _] = current_node->children.try_emplace(directory.string());
    current_node = &child->second;
  }

  current_node->is_terminal = true;

  return false;
}
```

### Example 2: A longer path below an existing one (`/a/b/c`)

In this case, the path `/a/b` is already stored in the trie and marked as terminal. When trying to insert `/a/b/c`, the function detects at node `b` that a complete path already ends here.

This means the new path lies underneath an existing one – **a clear conflict**.

```c++
namespace fs = std::filesystem;

auto has_conflict(path_node& root_node, fs::path const& path) -> bool {
  path_node* current_node = &root_node;

  for (auto const& directory : path) {
    if (current_node->is_terminal) { return true; }
    auto [child, _] = current_node->children.try_emplace(directory.string());
    current_node = &child->second;
  }

  current_node->is_terminal = true;

  return false;
}
```

### Example 3: The same path again (`/a/b`)

If the same path is inserted again, the function recognizes at the end that `b` is already marked as a terminal path. This is also treated as a conflict – **duplicate entries must be prevented**.

```c++
namespace fs = std::filesystem;

auto has_conflict(path_node& root_node, fs::path const& path) -> bool {
  path_node* current_node = &root_node;

  for (auto const& directory : path) {
    if (current_node->is_terminal) { return true; }
    auto [child, _] = current_node->children.try_emplace(directory.string());
    current_node = &child->second;
  }

  if (current_node->is_terminal) { 
    return true; 
  }
  
  current_node->is_terminal = true;

  return false;
}
```

### Example 4: A shorter path (`/a`) after `/a/b` already exists

Here, the function tries to insert `/a` even though `/a/b` is already present. This also results in a conflict, because `/a` is a prefix of an existing path. The function detects this by checking whether node `a` already has child nodes.

```c++
namespace fs = std::filesystem;
namespace rng = std::ranges;

auto has_conflict(path_node& root_node, fs::path const& path) -> bool {
  path_node* current_node = &root_node;

  for (auto const& directory : path) {
    if (current_node->is_terminal) { return true; }
    auto [child, _] = current_node->children.try_emplace(directory.string());
    current_node = &child->second;
  }

  if (current_node->is_terminal || !rng::empty(current_node->children)) { 
    return true; 
  }
  
  current_node->is_terminal = true;

  return false;
}
```

### The final version

```c++
namespace fs = std::filesystem;
namespace rng = std::ranges;

auto has_conflict(path_node& root_node, fs::path const& path) -> bool {
  path_node* current_node = &root_node;

  for (auto const& directory : path.lexically_normal()) {
    if (directory.empty()) { break; }
    if (current_node->is_terminal) { return true; }
    auto [child, _] = current_node->children.try_emplace(directory.string());
    current_node = &child->second;
  }

  if (current_node->is_terminal || !rng::empty(current_node->children)) { 
    return true; 
  }
  
  current_node->is_terminal = true;

  return false;
}
```

This version includes all necessary checks to ensure consistent and conflict-free handling of paths. Two additional details make the function more robust in practice:

#### **`lexically_normal()`**

This version uses `lexically_normal()`. The reason: paths like `a/./b/../b/c` contain components such as `.` (current directory) and `..` (parent directory), which are otherwise processed literally when iterated:

```
a
.
b
..
b
c
```

Without normalization, the path would not be recognized as what it actually is – namely `a/b/c`. The conflict check would become unreliable or incorrect.

`lexically_normal()` ensures that such paths are cleaned up before being processed.

#### **`directory.empty()`**

Another technical detail: paths like `/a/b/` end with a slash. This would result in an **empty component** when iterated:

```
a
b
""
```

The line

```c++
if (directory.empty()) { break; }
```

prevents such empty components from being added to the trie. Without this check, paths like `/a/b` and `/a/b/` would be treated as different, even though they refer to the same location.

## Conclusion

The `has_conflict` function is not a complex algorithm, but it solves a concrete problem reliably. The trie structure allows each path to be represented in a structured, component-based way, which makes it easy to detect conflicts early.

In my use case, the function has proven to be solid and flexible. It works regardless of whether a path is longer, shorter, or identical to an existing one – and it can easily be adapted to other scenarios where path overlaps need to be avoided.

> **Project Note:** The `has_conflict` function is part of `dropclone` – a C++ tool for automated directory synchronization.
>  
  The project is still under development and available here:
>
  [**↪ GitHub link to dropclone**](https://github.com/adamczapla/dropclone).
  {: .prompt-info }

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