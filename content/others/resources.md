+++
title = "Rust 中文资源导航"
template = "info-page.html"
path = "resources"
+++

![license-CC--BY--NC--SA--4.0-green](https://img.shields.io/badge/license-CC--BY--NC--SA--4.0-green)

这里是陈寒彤, 一位 Rust 爱好者和开发者.

<!-- toc -->

## 前言提要

Rust 作为现代编程语言, 其学习资料颇为丰富. 但遗憾的是, 这些资料大部分均无中文版本, 对于非英语母语者, 仅依靠翻译工具还是稍显吃力.

虽然已经有对官方文档及其他一些优秀文档的翻译工作 (如 [rustwiki.org](https://rustwiki.org/)), 但译作大部分均暂时或长期缺乏维护, 这对于高速发展的 Rust 语言来说是不够的.

## 资源目录

这里列出了我推荐的、我翻译的(或我参与翻译的) Rust 相关资料:

### 初学者指南

_本部分包含了一些适合 Rust 初学者的学习资源, 循序渐进地帮助你掌握 Rust 编程语言._

#### 入门必读

- [Rust 学习之旅 `tour.han.rs` (建设中)](https://tour.han.rs/)

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Ftour.han.rs%2Flatest-commit%2Fnext&query=%24.date&label=Last%20updated)

- [Rust 语言圣经 `course.rs` (我的启蒙作, 推荐)](https://course.rs/)

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fsunface%2Frust-course%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

#### 进阶必读

- [Rust 语言速查 `cheats.han.rs`](https://cheats.han.rs/)

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Fcheats.han.rs%2Flatest-commit%2Fmaster&query=%24.date&label=Last%20updated)

  ![Status](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Fcheats.han.rs%2Fbranch-infobar%2Fmaster&query=%24.refComparison.behind&prefix=The%20base%20commit%20is%20&suffix=%20commit(s)%20behind%20the%20official%20one&label=Status)

  一份适合全阶段 Rust 开发者的速查表, 常看常新.

- ...

### 官方文档

- [`The Rust Programming Language`](https://doc.rust-lang.org/book/) **Rust 程序设计语言**

  中英对照译文见: [`book.han.rs` (建设中)](https://book.han.rs/).

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Fbook.han.rs%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

  ![Status](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Fbook.han.rs%2Fbranch-infobar%2Fmain&query=%24.refComparison.behind&prefix=The%20base%20commit%20is%20&suffix=%20commit(s)%20behind%20the%20official%20one&label=Status)

  其他已有的还在积极更新的优秀译文:
  
  - [`kaisery/trpl-zh-cn`](https://kaisery.github.io/trpl-zh-cn/)
  
    ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2FKaiserY%2Ftrpl-zh-cn%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

- [`The Rust Reference`](https://doc.rust-lang.org/reference/index.html) **Rust 语言参考**

  中英对照译文见: [`reference.han.rs` (计划中)](https://reference.han.rs/).

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Freference.han.rs%2Flatest-commit%2Fmaster&query=%24.date&label=Last%20updated)

  ![Status](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Freference.han.rs%2Fbranch-infobar%2Fmaster&query=%24.refComparison.behind&prefix=The%20base%20commit%20is%20&suffix=%20commit(s)%20behind%20the%20official%20one&label=Status)

  有译文, 但是 2 年前已停止更新: [`rust-lang-cn/reference-cn`](https://rustwiki.org/zh-CN/reference/).

- [`The Rustonomicon`](https://doc.rust-lang.org/nomicon/) **Rust 死灵书**

  中英对照译文见: [`nomicon.han.rs` (建设中)](https://nomicon.han.rs/).

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Fnomicon.han.rs%2Flatest-commit%2Fmaster&query=%24.date&label=Last%20updated)

  ![Status](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Fnomicon.han.rs%2Fbranch-infobar%2Fmaster&query=%24.refComparison.behind&prefix=The%20base%20commit%20is%20&suffix=%20commit(s)%20behind%20the%20official%20one&label=Status)

  其他已有的还在积极更新的优秀译文:
  
  - [`nomicon-zh-Hans`](https://nomicon.purewhite.io/)

    ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Frust-lang-cn%2Fnomicon-zh-Hans%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

### TWiR (This Week in Rust) **Rust 语言周刊**

_[`This Week in Rust`](https://this-week-in-rust.org/) 是 Rust 社区的一份周报, 每周发布一次, 汇总了 Rust 社区的新闻、文章、工具、库等信息._

_精选的 Rust 更新: 及时了解 Rust 社区的活动、学习资源和最新发展._

- [TWiR 译文辑录 `twir.han.rs`](https://twir.han.rs/)

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Ftwir.han.rs%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

### 未归类

- [Rust 多语言编程指南 `rust-polyglot.han.rs` (建设中)](https://rust-polyglot.han.rs/)

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Frust-polyglot.han.rs%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

- [Rusty Book (锈书) `rusty.han.rs` (建设中)](https://rusty.han.rs/)

  收录实际开发过程可能用到的三方库以及一些实用的代码片段.

  ![Last updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fgithub.com%2Fhan-rs%2Frusty.han.rs%2Flatest-commit%2Fmain&query=%24.date&label=Last%20updated)

### 其他参考资料

TODO
