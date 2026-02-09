+++
title = "从零开始了解 TLS 1.3 系列笔记（一）—— 导语"
date = "2025-09-12"
updated = "2026-02-09"
# description = ""

[taxonomies]
tags = ["TLS", "Build a TLS 1.3 library from scratch"]

[extra]
toc = false
+++

## 导语

笔者一直认为, 个人是难以面面俱到地从零设计一个安全协议的: 密码学算法固然重要, 但错误的密码学实践与不安全的算法具有同等危害; 总有人试图自创加密协议, 但结果往往是造方形轮子乃至破轮子, 不如直接使用成熟的协议来得安全可靠. Transport Layer Security (TLS, *传输层安全*) 协议可谓现代互联网安全的基石, 经历数十年的发展, 在实践中缝缝补补、推陈出新, 如今来到了 1.3 版本. 出于实际需求, 笔者最近半年的空余时间都在阅读 [draft-ietf-tls-rfc8446bis] (即 [RFC8446]), 并结合 [rustls] 源码研究 TLS 1.3 实现. 苦于互联网关于 TLS 1.3 的中文内容相当零散, 遂开个新坑, 从零开始介绍(~~学习~~) TLS 1.3 协议的实现细节, 一方面方便笔者自己随时查阅, 另一方面也方便后来者参考学习, 减轻阅读晦涩的英文资料的烦恼.

## 本系列内容

本系列按照以下大纲进行撰写:

- 密码学应用基础.
- RFC 8446 规范详解.
- TLS 协议拓展, 包括但不限于:
  - ECH, 见 [draft-ietf-tls-esni].
- 实战: 从零开始, 实现一个简易但 Production Ready 的 TLS 1.3 加密库 (in Rust).

由于涉及的内容繁杂且笔者精力有限只能抽空闲时间学习、总结, 预计这个系列在一年内完成. 欢迎各位读者参考学习、批评指正. 感兴趣的读者可以点击 Tag 订阅 RSS.

## 参考文献

笔者在编写此系列文章的过程中学习参考了大量资料, 在此列出质量较高的部分, 以飨读者:

- [冰霜之地 (Halfrost)](https://halfrost.com/tag/https/) 分享的系列文章
- [超超哥哥 (taikulawo)](https://chaochaogege.com) 分享的文章
- [於清樂 (thiscute)](https://thiscute.world/) 分享的翻译文章
  - [Practical Cryptography for Developers Book](https://github.com/nakov/Practical-Cryptography-for-Developers-Book)
- ...

[rustls]: https://github.com/rustls/rustls
[RFC8446]: https://datatracker.ietf.org/doc/html/rfc8446
[draft-ietf-tls-rfc8446bis]: https://www.ietf.org/archive/id/draft-ietf-tls-rfc8446bis-14.html
[draft-ietf-tls-esni]: https://www.ietf.org/archive/id/draft-ietf-tls-esni-25.html
