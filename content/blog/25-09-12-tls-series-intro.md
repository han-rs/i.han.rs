+++
title = "从零开始了解 TLS 1.3 系列笔记（一）—— 导语"
date = "2025-09-12"
# updated = ""
# description = ""

[taxonomies]
tags = ["TLS"]

[extra]
toc = true
+++

## 导语

Transport Layer Security (TLS, *传输层安全*) 可谓现代互联网安全的基石, 经历数十年的发展, 在实践中缝缝补补、推陈出新, 如今来到了 1.3 版本.
笔者一直坚定认为, 个人难以面面俱到地从零设计一个安全协议: 密码学算法固然重要, 但错误的密码学实践与不安全的算法是同等危害的.
注意到总有人试图自创加密协议, 但往往是搬起石头砸自己的脚, 反而不如直接使用成熟的协议来得安全可靠.

出于实际需求, 笔者最近半年的空余时间都在阅读 [RFC8446] / [draft-ietf-tls-rfc8446bis], 结合 [rustls] 源码研究 TLS 1.3 实现.
苦于中文互联网关于 TLS 1.3 的内容相当零散, 遂开个新坑, 从零开始介绍(~~学习~~) TLS 1.3 协议的方方面面, 一方面方便自己随时查阅, 另一方面也方便后来者参考学习.

## 本系列内容

本系列内容主要包括以下部分:

- 密码学应用基础.
- 系列 RFC 翻译、精读.
- 参考 [rustls], 从零开始实现一个简易的 TLS 1.3 库.
- 拓展部分
  - ECH, 见 [draft-ietf-tls-esni]

内容繁杂, 预计一年内完成, 欢迎各位读者批评指正.

## 参考资料及致谢

笔者在学习过程中参考了大量资料, 在此一并列出, 以飨读者:

- [halfrost 的系列文章](https://halfrost.com/tag/https/)
- [超超哥哥的系列文章](https://chaochaogege.com)
- ...

[rustls]: https://github.com/rustls/rustls
[RFC8446]: https://datatracker.ietf.org/doc/html/rfc8446
[draft-ietf-tls-rfc8446bis]: https://tlswg.org/tls13-spec/draft-ietf-tls-rfc8446bis-13/draft-ietf-tls-rfc8446bis.html
[draft-ietf-tls-esni]: https://tlswg.org/draft-ietf-tls-esni/#go.draft-ietf-tls-esni.html
