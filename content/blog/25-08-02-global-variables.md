+++
title = "使用 Rust 全局变量的注意事项"
date = "2025-08-02"
updated = "2025-09-14"
# description = ""

[taxonomies]
tags = ["Rust"]

[extra]
# toc = true
tldr = """
在编写 Rust 库 (library) 时, 应当慎用全局变量. 当应用使用了某些上层库, 而这些上层库意外地引用了不同版本的当前的库时, 会导致全局变量出现 "多个实例" 的问题.
"""
+++

应当区分 `static` 和 `const` 关键字的作用, 正如其[文档](https://doc.rust-lang.org/std/keyword.const.html)所言:

> A static item is a value which is valid for the entire duration of your program (a `'static` lifetime).
>
> 静态变量是一个在程序的整个生命周期内有效的值 (拥有 `'static` 的生命周期).

> `const` items look remarkably similar to `static` items, which introduces some confusion as to which one should be used at which times. To put it simply, constants are inlined wherever they’re used, making using them identical to simply replacing the name of the `const` with its value. Static variables, on the other hand, point to a single location in memory, which all accesses share. This means that, unlike with constants, they can’t have destructors, and act as a single value across the entire codebase.
>
> `const` 与 `static` 看起来非常相似, 可能在使用上引起困惑. 简单来说, 常量在使用的地方都会被内联, 这使得使用它们就像简单地用其值替换 `const` 的名称一样. 另一方面, 静态变量指向内存中的一个单一位置, 所有访问都共享该位置. 这意味着, 与常量不同, 它们不能有析构函数, 并且在整个代码库中充当一个单一值.

参考资料:

- [reddit](https://www.reddit.com/r/rust/comments/13lsdq8/global_statics_from_libraries/)
