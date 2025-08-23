+++
title = "This Week In Rust (#613) 速报"
date = "2025-08-23"
# updated = ""
# description = "This Week In Rust (#613) 速报"

[taxonomies]
tags = ["Rust"]

[extra]
toc = true
+++

最新一期 This Week In Rust (#613) 新鲜出炉: [This Week in Rust #613](https://this-week-in-rust.org/blog/2025/08/20/this-week-in-rust-613/).

## Updates from Rust Community

### Official

- [Demoting x86_64-apple-darwin to Tier 2 with host tools](https://blog.rust-lang.org/2025/08/19/demoting-x86-64-apple-darwin-to-tier-2-with-host-tools/)

  *评: 光阴似箭, M1 已经是五年前的产物, 在苹果的强力推动下, MacOS 生态对 ARM 架构的支持已经相当完善. 笔者表示, 不愧是苹果.*
  
  *仍然在使用搭载 Intel 处理器的 x86_64 架构的苹果电脑的人越来越少, 自然也就难以继续维持 Tier 1 的支持了.*

- Leadership Council September 2025 Representative Selections
- Electing new Project Directors 2025

### Newsletters

- This Month in Rust OSDev: July 2025
- The Embedded Rustacean Issue #52

## Project/Tooling Updates

- [Zed for Windows: What's Taking So Long?!](https://zed.dev/blog/windows-progress-report)

  *评: 笔者表示, 已经受不了 VSCode 动辄几个 GB 的内存占用了... 期待 Zed 能够早日发布 Windows 版本.*

- SeaQuery just made writing raw SQL more enjoyable (*数据库*)
- r3bl-cmdr v0.0.22 (*TUI*)
- r3bl_tui v0.7.4 (*TUI*)
- Heapless v0.9.1 - static friendly data structures that don't require dynamic memory allocation (*嵌入式开发*)
- [Announcing Asterinas 0.16.0](https://asterinas.github.io/2025/08/04/announcing-asterinas-0.16.0.html) (*操作系统内核, 仓库创始人是国人来着, 文档有中文, 泪目*)

## Observations/Thoughts

- [Placing Arguments](https://blog.yoshuawuyts.com/placing-arguments/)

  *此文介绍了可能在未来引入标准库的 `#[placing]` 属性的 API 设计. `#[placing]` 主要是为了解决实现返回类型在调用方的堆栈帧中构造的问题.*

- [Update on our advocacy for memory-safety - Tweede golf](https://tweedegolf.nl/en/blog/160/update-on-our-advocacy-for-memory-safety)

  *TL, DR: please use Rust for memory safety!*

- [Speed wins when fuzzing Rust code with #[derive(Arbitrary)]](https://nnethercote.github.io/2025/08/16/speed-wins-when-fuzzing-rust-code-with-derive-arbitrary.html)

  *TL, DR: 请升级 `arbitrary` 到 1.4.2, 性能大提升!*

- [Rewriting Numaflow’s Data Plane: A Foundation for the Future](https://blog.numaproj.io/rewriting-numaflows-data-plane-a-foundation-for-the-future-a64fd2470cf0)

  *Numaflow 是一个开源 K8s 原生平台, 将使用 Rust 重构控制平面*

  *其实国内也有个中科院软研所主办的 [R2CN（RISC-V Rust for Cloud Native）开源实习计划](https://r2cn.dev/), 旨在推动 RISC-V 生态和 Rust 在云原生领域的应用, 仓库是 [rk8s](https://github.com/r2cn-dev/rk8s).*

- Terminal sessions you can bookmark: Building Zellij's web client

  *Zellij is a workspace aimed at developers, ops-oriented people and anyone who loves the terminal. Similar programs are sometimes called "Terminal Multiplexers".*

- [Testing failure modes using error injection](https://forgestream.idverse.com/blog/20250814-testing-failure-modes/)

  *使用错误注入测试故障模式*

- [Multiple Breakpoints in Rust: Ownership-Driven Debugger Design](https://system.joekain.com/2025/08/17/ownership-driven-debugger-design.html)

  *对调试器开发感兴趣的可以看看*

- [Lessons learned from rewriting the UltraGraph crate](https://deepcausality.com/blog/lessons-learned-from-rewriting-ultragraph/)
- [Scientific Computing in Rust](https://ideas.reify.ing/en/blog/scientific-computing-in-rust-with-pytorch/)

  *Rust 做科学计算么, 有意思*

- [RKL: A Docker-like Command-line Interface Built in Rust](https://r2cn.dev/blog/rkl-a-docker-like-command-line-interface-built-in-rust)

  *巧了, 刚才才提到 R2CN. ~~就是, 得把国人用英文写的文章再翻回去么, 有意思~~*

- [kruci: Post-mortem of a UI library](https://pwy.io/posts/kruci-post-mortem/)

  *~~TL, DR: 嫌弃 rataui 性能不行, 作者重写了一个~~ 感兴趣的可以尝鲜.*

- [Nine Rules for Generalizing Your Rust Library: Lessons from Extending RangeSetBlaze to Maps (Part 2)](https://medium.com/@carlmkadie/nine-rules-for-generalizing-your-rust-library-part-2-92bb899d47ef)

  *好文, 已加入 TWIR 翻译计划 (咕咕咕)*

- [audio] Intrusive lists for fun and profit

## Rust Walkthroughs

- [Constructor Best Practices in Rust](https://blog.cuongle.dev/p/constructor-best-practices-in-rust)

  *好文, 已加入 TWIR 翻译计划 (咕咕咕)*

- [Let's write a macro in Rust - Part 1](https://hackeryarn.com/post/rust-macros-1/)

  *好文, 已加入 TWIR 翻译计划 (咕咕咕)*

- [Memory analysis in Rust](https://rumcajs.dev/posts/memory-analysis-in-rust/)

  *好文! 已加入 TWIR 翻译计划 (咕咕咕)*

## Miscellaneous

- Rust At Microsoft And Chairing The Rust Foundation
- Talking To Zed Industries- Makers Of The 100% Rust, Super-Performant, Collaborative Code Editor
- [All the Rust Tutorials](https://seanborg.tech/blog/huge-tutorial-list/)

  *收录了 Rust 开发各种 tutorial, 就是网站略为卡顿.*

- July 2025 Rust Jobs Report

## Call for Participation; projects and speakers

略.

## Updates from the Rust Project

简要列举几个我觉得很实用的:

- [implement `#[derive(From)]`](https://github.com/rust-lang/rust/pull/144922)

  将于 Rust 1.91.0 推出. 说实在的, 不知道为什么拖了这么久.

- [add Default impls for `Pin`ned `Box`, `Rc`, `Arc`](https://github.com/rust-lang/rust/pull/143717)

  将于 Rust 1.91.0 推出. 也是比较常用的了.

- [stabilize `#![feature(ip_from)]`](https://github.com/rust-lang/rust/pull/141744)

  将于 Rust 1.91.0 推出. 这个也是咕了很久了, 还好我习惯使用 nightly Rust. 不知道为什么拖了这么久.

## Upcoming Events

略.
