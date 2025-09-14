+++
title = "Notes on Rust's `static` Variables"
date = "2025-08-02"
updated = "2025-09-14"
# description = ""

[taxonomies]
tags = ["Rust"]

[extra]
# toc = true
tldr = """
When writing a Rust library, global variables should be used with caution. When an application depends on certain upstream crates, while these upstream crates inadvertently depends on different versions of one crate, we would get "multiple instances" of global variables.
"""
+++

The roles of the `static` and `const` keywords should be distinguished, as stated in the [documentation](https://doc.rust-lang.org/std/keyword.const.html):

> A static item is a value which is valid for the entire duration of your program (a `'static` lifetime).

> `const` items look remarkably similar to `static` items, which introduces some confusion as to which one should be used at which times. To put it simply, constants are inlined wherever they’re used, making using them identical to simply replacing the name of the `const` with its value. Static variables, on the other hand, point to a single location in memory, which all accesses share. This means that, unlike with constants, they can’t have destructors, and act as a single value across the entire codebase.

References:

- [reddit](https://www.reddit.com/r/rust/comments/13lsdq8/global_statics_from_libraries/)
