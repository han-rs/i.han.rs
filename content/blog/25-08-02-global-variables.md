+++
title = "随笔一则"
date = "2025-08-02"
# updated = ""
description = "Meme: Rust 全局变量"

[taxonomies]
tags = ["Rust"]

[extra]
# toc = true
tldr = """
在编写 Rust 库 (library) 时, 应当慎用全局变量. 当应用使用了某些上层库, 而这些上层库意外地引用了不同版本的当前的库时, 会导致全局变量出现 "多个实例" 的问题.
"""
+++

参考资料:

- [reddit](https://www.reddit.com/r/rust/comments/13lsdq8/global_statics_from_libraries/)
