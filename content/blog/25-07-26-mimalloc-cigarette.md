+++
title = "译: Mimalloc Cigarette: Losing one week of my life catching a memory leak"
date = "2025-07-26"
# updated = ""
description = "Mimalloc 香烟: 我因内存泄漏而失去了生命中的一周"

[taxonomies]
tags = ["翻译", "Rust", "mimalloc"]

[extra]
# toc = true
tldr = """
⭐ 译者总结

使用 `mimalloc` 时, 被标记为 "已释放" 的内存只会在线程活跃时, 如执行有其他内存分配操作时, 被整理 (bookkeeping)、释放. 换句话说, **线程休眠期间不会释放先前分配但不再使用了的内存**.
"""
+++

> 本文翻译自 [Mimalloc Cigarette: Losing one week of my life catching a memory leak](https://pwy.io/posts/mimalloc-cigarette/), 英文原文版权由原作者所有, 中文翻译版权遵照 CC BY-NC-SA 协议开放.
>
> 囿于译者自身水平, 译文虽已力求准确, 但仍可能词不达意, 欢迎批评指正.
>
> 2025 年 7 月 26 日晨, 于广州.

<details>
    <summary>目录</summary>
    <!-- toc -->
</details>

One of applications at my work has always been RAM-bound - it's a pricing engine that basically loads tons of hotels, builds some in-memory indices and then allows you to issue queries like find me the cheapest hotel in berlin, pronto.

我工作中的一个应用程序一直受到 RAM 限制: 它是一个定价引擎, 基本上是加载大量酒店数据, 构建一些内存索引, 然后允许你发出查询, 比如快速找到柏林最便宜的酒店.

A pricing engine's main purpose is to price engines hotels, but in order to do that effectively, there's a lot of "meta work" involved, like:

定价引擎的主要目的是为酒店定价, 但为了有效地做到这一点, 涉及很多"元工作", 比如：

- Where do we load the data from, how do we do it?

  我们从哪里加载数据, 如何加载？

- Do we load the entire dataset or just some parts of it?

  我们是加载整个数据集还是只加载其中的一部分？

- Should we precalculate prices in order to speed-up the most popular queries?

  我们是否应该预先计算价格以加速最受欢迎的查询？

Such an engine poses an interesting technical challenge, even greater so when one day it starts OOMing on production, even though the entire dataset should fit in the memory multiple times...

这样的引擎带来了有趣的技术挑战, 当有一天它在生产环境中开始出现内存不足时, 挑战变得更大, 尽管整个数据集应该能在内存中放下多次...

## Hello, World!

Simplifying ~~a bit~~ a lot, what we're dealing with is:

~~简要~~大大简化一下, 我们处理的是：

```rust
struct State {
    hotels: Vec<Hotel>,
}

struct Hotel {
    prices: Vec<f32>,
}

// ---

impl State {
    fn new() -> Self {
        Self {
            hotels: load_hotels().collect(),
        }
    }

    fn refresh(&mut self) {
        for (id, hotel) in load_hotels().enumerate() {
            self.hotels[id] = hotel;
        }
    }
}

fn load_hotels() -> impl Iterator<Item = Hotel> {
    (0..1_000).map(|_| Hotel {
        prices: (0..1_000_000).map(|n| n as f32).collect(),
    })
}

// ---

fn main() {
    let state = Arc::new(RwLock::new(State::new()));

    // Spawn a separate thread responsible for checking which hotels have
    // changed and need to be refreshed etc.
    thread::spawn({
        let state = Arc::clone(&state);

        move || loop {
            thread::sleep(Duration::from_secs(5));
            state.write().unwrap().refresh();
        }
    });

    // Now in reality we start a Rocket server here, but for practical
    // purposes stalling the main thread will be sufficient.
    loop {
        thread::sleep(Duration::from_secs(1));
    }
}
```

Of course IRL there's `ArcSwap` instead of `RwLock`, every hotel contains much more information (like taxes, discounts or supplements) etc., but we've got a reasonably good approximation here.

当然在现实生活中有 `ArcSwap` 而不是 `RwLock`, 每个酒店包含更多信息（如税费、折扣或附加费）等, 但我们在这里有一个相当好的近似.

Now, what if I told you that this code has totally different memory characteristics depending on which allocator you use?

现在, 如果我告诉你这段代码根据你使用的分配器有完全不同的内存特性, 你会怎么想？

## Practical Reasons | 原因分析

Memory allocator is the piece of software invoked whenever your program needs to get hands on extra memory, like when you're calling `Box::new()`. And memory allocation is quite a complex topic, with different solutions offering different trade-offs.

内存分配器是当你的程序需要获得额外内存时调用的软件, 比如当你调用 `Box::new()` 时. 内存分配是一个相当复杂的话题, 不同的解决方案提供不同的权衡.

Say, when implementing a firmware you might pick an allocator that works slower, because its implementation is just simpler and doesn't occupy much space in the final binary.

比如说, 在实现固件时, 你可能会选择一个工作较慢的分配器, 因为它的实现更简单, 在最终二进制文件中不占用太多空间.

(some would argue that embeddeed programs shouldn't allocate, yadda yadda, but you get the idea -- replace firmware with wasm and you end up with the same problem.)

（有些人会争论说嵌入式程序不应该分配内存, 等等等等, 但你懂我意思: 把固件替换为 WASM, 你最终会遇到同样的问题. ）

`mimalloc` is an allocator that fights tooth and nail for performance - and while most applications don't have to worry about allocation performance, in our case internal benchmarks have proven it gives us extra 10% for free; when your response times have to be within milliseconds, this matters.

`mimalloc` 是一个为性能而生的分配器: 虽然大多数应用程序不必担心分配性能, 但在我们的情况下, 内部基准测试证明它无需额外配置即给我们带来额外的 10% 性能提升；当你的响应时间必须在毫秒内时, 这很重要.

But there's a catch.

但有一个陷阱.

## Désenchantée | 失望

That program from before, on my x86_64 Linux machine it allocates around 4 GB of memory and it remains on this level through the refreshing.

之前的那个程序, 在我的 x86_64 Linux 机器上分配大约 4 GB 内存, 并在刷新过程中保持在这个水平.

This makes sense, right? We're using lazy iterators, replacing stuff in place one-by-one, there's no reason we'd need more RAM.

这是有道理的, 对吧？我们使用惰性迭代器, 逐个地就地替换东西, 没有理由需要更多 RAM.

But if you use `mimalloc`:

但如果你使用 `mimalloc`：

```rust
use mimalloc::MiMalloc;

#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;
```

... the program will first allocate 4 GB and then allocate extra 4 GB during the refreshing, oh noes!

... 程序将首先分配 4 GB, 然后在刷新期间分配额外的 4 GB, 哦 f***!

Getting to this point already took three days of my life - believe me or not, when faced with 200k lines of Arc-ridden Rust code that seems to generate a memory leak, one's first thought is not "let's try with different allocator", but rather "probably something's holding onto an Arc for too long".

确认这一点已经花费了我三天的生命: 信不信由你, 当面对 20 万行充满 Arc 的 Rust 代码似乎产生内存泄漏时, 人们的第一个想法不是 "让我们尝试不同的分配器", 而是 "可能有什么东西持有 Arc 太久了".

And so I've valgrind-ed. I've perf-ed. I've analyzed assembly. I've headbanged and I've cried.

我用了 `valgrind`. 我用了 `perf`. 我分析了汇编. 我撞头, 我哭.

No more.

不再这样了.

From now on I'm always assuming it's someone else's fault - it's the allocator, it's the compiler, it's that crate Mike pulled last night. WHY DO YOU HATE ME MIKE, WHY ARE YOU PULLING RANDOM CRATES TO MY PURE ~~~tv noise~~~

从现在开始, 我将假设这是别人的错: 是分配器的错, 是编译器的错, 是 Mike 昨晚拉取的那个 crate 的错. (碎碎念)

## Remède | 解决方案

Allocators have different characteristics for a reason - they do some things differently between each other. What do you think `mimalloc` does that could account for this behavior?

分配器有不同的特性是有原因的: 它们彼此之间做一些不同的事情. 你认为 `mimalloc` 做了什么可能导致这种行为？

Let me give you two hints, two pieces of code that solve the problem, but feel cursed:

让我给你两个提示, 两段解决问题但感觉被诅咒的代码：

```rust
// Approach 1:
fn main() {
    let state = thread::spawn(|| {
        Arc::new(RwLock::new(State::new()))
    }).join().unwrap();

    /* ... */
}

// Approach 2:
fn main() {
    /* ... */

    loop {
        Box::new(1234);
    }
}
```

Any ideas? Last chance to win a plushie polar bear!

有什么想法吗？赢得毛绒北极熊的最后机会!

Alright then, the issue is that mimalloc assumes that every thread allocates every now and then.

好吧, 问题是 `mimalloc` 假设每个线程时不时地分配内存.

Every now and then during `malloc()`, `mimalloc` performs some internal bookkeeping, so when a thread goes to sleep (say, because it delegates handling HTTP requests into a separate thread pool...), this bookkeeping doesn't happen (for that particular thread).

在 `malloc()` 期间, `mimalloc` 时不时地执行一些内部整理, 所以当一个线程进入睡眠状态时（比如, 它将处理 HTTP 请求委托给一个单独的线程池...）, 这种整理不会发生（对于那个特定的线程）.

The most nasty edge case that can happen here, and the one that we've stumbled upon, is when your thread allocates a lot of data, then launches other threads to work on that data, and then goes to sleep. As other threads work on memory and override stuff, Rust destructors are launched properly, but the underlying memory blocks simply get marked as "to be released".

这里可能发生的最讨厌的边缘情况, 也是我们遇到的情况, 是当你的线程分配大量数据, 然后启动其他线程来处理这些数据, 然后进入睡眠状态. 当其他线程处理内存并覆盖东西时, Rust 析构函数被正确启动, 但底层内存块只是被标记为 "待释放".

Under normal conditions, these blocks get processed a moment later, during a call to `malloc()` on the thread that created them - but if that thread is asleep, those blocks never become available again (unless the thread dies, of course).

在正常条件下, 这些块会在稍后处理, 在创建它们的线程调用 `malloc()` 期间. 但如果那个线程在睡眠, 这些块永远不会再次可用（当然, 除非线程终止）.

To be sure, the problem is not that the memory is not returned to the kernel - that's alright. It's that unless this bookkeeping happens, mimalloc can't even use the memory for itself - all this free estate just lays there, dormant:

可以肯定的是, 问题不是内存没有返回给内核, 那没关系. 问题是除非这种整理发生, 否则 `mimalloc` 甚至不能为自己使用内存: 所有这些空闲空间就躺在那里, 休眠：

- <https://github.com/microsoft/mimalloc/issues/537>
- <https://github.com/microsoft/mimalloc/issues/214>

Anyway, the solution we went with was to keep all refreshing on the same thread - when program starts, we spawn a dedicated refreshing-thread and use channels to let it know to do its thing.

无论如何, 我们采用的解决方案是将所有刷新保持在同一个线程上: 当程序启动时, 我们生成一个专用的刷新线程, 并使用通道让它知道要做什么.

So yeah, that was fun; and health-wise probably more like seven cigarettes.

所以是的, 那很有趣；从健康角度来说, 可能更像七支香烟.

---

> 推荐阅读: <https://www.cnblogs.com/linkwk7/p/11221730.html>
