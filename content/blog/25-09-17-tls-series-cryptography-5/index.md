+++
title = "从零开始了解 TLS 1.3 系列笔记 (六) —— 实用密码学之「对称加密与非对称密码学」"
date = "2025-09-17"
updated = "2025-09-18"
description = "本文介绍了对称加密和非对称密码学, 重点介绍了 AEAD 加密方案, 以及公钥密码系统在数字签名领域的应用. 最后补充介绍了数字证书相关内容."

[taxonomies]
tags = ["TLS", "Cryptography"]

[extra]
toc = true
katex = true
tldr = """
1. AES-128-GCM, AES-256-GCM 和 ChaCha20-Poly1305 是现代应用程序应该优先选用的对称加密方案, 它们是 AEAD 加密方案. 这些对称加密方案的密文长度和明文长度相关, 必要时应当进行填充 (padding) 以掩盖明文的真实长度.
1. 分组模式 (如 GCM, CTR) 可以将分组密码算法 (如 AES) 转换为流密码算法, 以加密任意长度的数据. 使用到的 IV (nonce) 用以防止加密相同内容得到相同密文, 所以绝不允许重用.
1. 基于公钥加密系统的非对称加密方案计算复杂且只能加密/解密相对短的消息. 现代密码学实践中, 一般混合使用对称加密和非对称加密, 以发挥各自的优势.
1. RSA 和 ECC 是两种常见的公钥密码系统, 基于后者的数字签名算法 (如 ECDSA, EdDSA) 由于相同加密强度下密钥更短且计算更快, 正在逐渐取代前者.
1. 推荐使用 Curve25519 / Ed25519 和 Curve448 / Ed448 等现代椭圆曲线, 避免使用 NIST 推荐的 P-256, P-384 等曲线. 相对应的密钥交换算法是 X25519 和 X448; 数字签名算法是 Ed25519 和 Ed448 (统称 EdDSA).
"""
+++

> 一些密码学高频英文术语需要我们了解
>
> 1. 加密 (encrypt), 解密 (decrypt)
> 1. 密文 (ciphertext), 明文 (plaintext)
> 1. 对称加密 (symmetric encryption), 非对称加密 (asymmetric encryption)
> 1. "cipher": 一般指 "加密算法", 但有些翻译会直译为 "密码", 需要和 "password" 区分.

密码学中, 依据加解密密钥是否相同, 可以将加密算法分为对称加密和非对称加密两大类.

## 对称加密

对称加密, 即加密和解密使用相同的密钥. 这种加密方式的优点是加解密速度快, 适合处理大量数据. 常见的对称加密算法包括 AES 和 Chacha20 等.
其中绝大多数都是 "块密码算法 (Block Cipher)" 或者叫 "**分组密码算法**", 这种算法**一次只能加密固定大小的块** (例如 128 位), 少部分是 "**流密码算法** (Stream Cipher)", 流密码算法允许将数据逐字节地加密为密文流.
可以使用称为*分组密码工作模式*的技术, 让分组密码算法也能做到流式加密, 我们会在后文详细了解.

用于加密/解密的密钥也被称为共享密钥(需要在通信双方之间共享). 可以是 PSK, 如从密码利用 KDF 派生得到, 或编码为 Base58 / Base64 格式; 或者通过密钥交换方案协商或传输.

> 小贴士
>
> 对称加密所使用的密钥一般是 128 / 192 / 256 位的二进制数据, 直接使用不便于存储和传输, 因此通常会编码为 Base58 / Base64 格式.
>
> 原始 (按 HEX 格式给出):
>
> ```
> 02c324648931b89e3e8a0fc42c96e8e3be2e42812986573a40d46563bceaf75110
> ```
>
> Base58 编码:
>
> ```
> pbPRqYDxnKZfs8j4KKiqYmx6nzipAjTJf1oCD1WKgy99
> ```
>
> Base64 编码:
>
> ```
> AsMkZIkxuJ4+ig/ELJbo474uQoEphlc6QNRlY7zq91EQ
> ```

大多数现代对称密钥密码算法都是抗量子的 (quantum-resistant), 当使用长度足够的密钥时 (如 AES-256), 即便是量子计算机也无法破坏其安全性.

### 对称加密方案

我们知道, 单纯使用加密算法只能保证数据的安全性, 并不能满足我们对消息真实性、完整性与不可否认性的需求, 因此通常我们会将对称加密算法跟其他算法组合成一个对称加密方案来使用:

1. KDF (如果是密码作 PSK 的模式, 需要使用 KDF 从密码学强度低的密码派生出密码学强度高的伪随机密钥)
1. 分组密码工作模式 (如 CBC 或 CTR) + 消息填充算法 (如 PKCS7): 分组密码算法, 如 AES, 需要借助这两种算法, 才能加密任意大小的数据 (即所谓的数据流). 而一个流密码加密方案本身就能加密任意长度的数据, 因此不需要分组密码模式与消息填充算法.
1. 密码算法.
1. 消息认证算法: 如 HMAC, 用于验证消息的真实性、完整性、不可否认性.

如 AES-256-CTR-HMAC-SHA256 就表示一个使用 AES-256 与 Counter 分组模式 (CTR) 进行加密, 使用 HMAC-SHA256 进行消息认证的加密方案.

### 分组密码工作模式

前面我们提到分组密码工作模式可以让分组密码算法也能加密任意长度的数据. 其中涉及一些概念值得我们了解.

加密方案的名称中就带有具体的分组模式名称, 如:

- AES-256-GCM - AES 密码算法 + 256 位加密密钥 + Galois Counter Mode (GCM) 分组模式
- AES-128-CTR - AES 密码算法 + 128 位加密密钥 + Counter (CTR) 分组模式

分组密码工作模式背后的主要思想是把明文分成多个长度固定的组, 再在这些分组上重复应用分组密码算法进行加密/解密, 以实现安全地加密/解密任意长度的数据.
某些分组模式 (如 CBC) 要求将输入拆分为分组, 并使用填充算法将最末尾的分组填充到块大小, 也有些分组模式 (如 CTR、CFB、OFB、CCM、EAX 和 GCM) 不需要, 因为它们在每个步骤中, 都直接在明文部分和内部密码状态之间执行异或 (XOR) 运算.

使用 "分组模式" 加密大量数据的流程基本如下:

1. 使用加密密钥 + IV (Initialization Vector, 初始向量) 初始化加密算法状态
1. 加密数据的第一个分组
1. 使用加密密钥和其他参数转换加密算法的当前状态
1. 加密下一个分组
1. 再次转换加密状态

依此类推, 直到处理完所有输入数据. 解密的流程类似.

值得注意的是, 显然, 使用 CTR / GCM 分组模式时, **密文的大小与明文相同**, 必要时应当进行填充 (padding) 以掩盖明文的真实长度.

#### IV

IV 也被称作 salt 或者 **nonce**, 通常是一个随机数, 主要作用是往密文中添加随机性, 使同样的明文被多次加密也会产生不同的密文, 从而确保密文的不可预测性.

IV 的大小应与密码块大小相同.

IV 通常无需保密, 但应当足够随机 (GCM 模式下除外). 绝不允许重用 (nonce 即 number once, 一次性的数字).

#### GCM 分组模式

GCM (Galois Counter Mode) 是一种广泛使用的分组密码工作模式, 结合了 CTR 模式的加密和 Galois 域上的**消息认证**. GCM 模式下的 AES 加密过程除了得到密文外, 还会得到一个**认证标签** (authentication tag), 一般附在密文后面.

### AE(AD)

AE(AD) 即 Authenticated Encryption (with Associated Data), (带关联数据的)认证加密, 我们前面已经提到过这个名词, 其包含了 "认证" 和 "加密" 两大部分.

1. 认证 (Authentication): 通常使用 HMAC, Poly1305, 或者 GCM 分组模式自带.
2. 加密 (Encryption): 使用对称加密算法, 如 AES 或 ChaCha.

正确实现了消息认证的对称加密方案, 如 AES-256-CTR-HMAC-SHA256, AES-128-GCM, AES-256-GCM 或 ChaCha20-Poly1305 等, 均可称为 AE(AD) 加密方案. 今天的大多数应用程序应该优先选用 AEAD 加密方案进行对称加密, 而不是自己造轮子, 由于密码学的复杂性, 自己造轮子往往会出错.

> 性能小贴士: AES 和 Chacha 怎么选择?
>
> 1. 机器 CPU 支持 AES 硬件加速 (大部分现代 CPU 都支持), 使用 AES.
> 1. 机器 CPU 不支持 AES 硬件加速, 如 2014 年以前的 x86 架构处理器, 或者移动端相当一部分 ARM 架构处理器, 或者路由器所使用的大部分的 MIPS 架构处理器. 此时使用 ChaCha 会更快 (本来就是为移动端特别优化的).
> 1. 如果机器 CPU 足够现代且性能强劲, AES 和 ChaCha 差别不大, 在 HTTPS 场景下基本不可能是加密能力瓶颈, 任选其一即可.
>
> 如果想测试, 可以尝试运行:
>
> ```bash
> openssl speed -evp aes-128-gcm
> openssl speed -evp chacha20-poly1305
> ```
>
> 参考输出:
>
> ```bash
> > openssl speed -evp chacha20-poly1305
> ... (略) ...
> The 'numbers' are in 1000s of bytes per second processed.
> type              2 bytes     31 bytes    136 bytes   1024 bytes   8192 bytes  16384 bytes
> ChaCha20-Poly1305   135090.41k   398677.07k   685502.68k  4633523.20k  5013787.99k  5053565.35k
> > openssl speed -evp aes-128-gcm
> ... (略) ...
> The 'numbers' are in 1000s of bytes per second processed.
> type              2 bytes     31 bytes    136 bytes   1024 bytes   8192 bytes  16384 bytes
> AES-128-GCM      16855.33k   246339.19k  1061738.45k  4942513.83k 13130011.57k 14806788.78k
> ```
>
> 结果显示, 在笔者的机器上, AES-128-GCM 的加密速度 (14.8 GB/s) 明显快于 ChaCha20-Poly1305 (5.1 GB/s).
> 然而, 5 GB/s 是 40 Gbps 的网络带宽才能达到的速度了, 还只用到了一个核心.

### 对称加密算法

以下列举一些常见的对称加密算法.

安全的:

- AES (Advanced Encryption Standard, 高级加密标准): 目前最流行的对称加密算法, 由 NIST 于 2001 年发布. 支持 128, 192 和 256 位密钥长度. AES 是一种分组密码算法, 块大小为 128 位, 使用 128 位的 IV. AES-GCM 是目前最常用的对称加密方案之一.
- ChaCha: Salsa 改进版变种, 包括 Chacha20 等. 使用 128 / 256 位密钥和 64 位 IV. ChaCha20-Poly1305 也是一种流行的对称加密方案, 结合了 ChaCha20 流密码和 Poly1305 消息认证码.
- 其他: 包括但不限于 RC5, RC6, 韩国的 ARIA (类似 AES), 中国的 SM4 (类似 AES) 等.

由于出现安全问题已弃用的:

- DES
- 3DES
- RC4
- Blowfish
- ...

---

> 在介绍完对称加密后, 我们来了解一下非对称加密. 在此之前, 我们先了解一下非对称密码学(公钥密码学)的基本概念.

## 公钥密码学

1. 公钥密码系统的密钥始终以公钥/私钥对(**密钥对**, Key Pair)的形式出现, 公钥密码系统提供数学框架和算法来生成密钥对. 公钥通常与所有人共享, 而私钥则保密. 公钥密码系统在设计时就确保了在预期的算力下, 几乎不可能从其公开的公钥逆向演算出对应的私钥.
1. 公钥密码系统主要有三大用途: **加密与解密**、**签名与验证**、**密钥交换**. 每种算法都需要使用到公钥和私钥, 比如由公钥加密的消息只能由私钥解密, 由私钥签名的消息需要用公钥验证. 由于加密解密、签名验证均需要两个不同的密钥, 故公钥密码学也被称为非对称密码学.

比较著名的公钥密码系统有: RSA、ECC、(EC)DH. 不同的公钥密码系统可以提供以下一项或多项功能:

- 执行密钥对生成: 随机生成私钥和对应的公钥;
- 执行加解密: 通过公钥加密, 通过私钥解密;
- 作为密钥交换算法;
- 执行数字签名 (消息认证): 通过私钥对消息进行签名, 通过公钥验证签名.

公钥密码系统的密钥交换功能, 我们已经在前面的文章讲述过, 在此不再赘述; 公钥密码系统的数字签名功能, 我们会在后文单独介绍.

### 非对称加密

应用公钥密码系统的加(解)密功能即称非对称加(解)密. 非对称加密安全性相对于对称加密高, 但缺点也明显:

- 算法更为复杂, 加解密速度比对称加密慢非常多.
- 只能加密/解密很短的消息.

  如在 RSA 系统中, 输入消息需要被转换为大整数, 例如使用 OAEP 填充, 然后才能被加密为密文. (密文实质上就是另一个大整数.)

  一些非对称密码系统, 如 ECC, 不直接提供加密能力, 需要结合使用更复杂的方案才能实现加解密.

为此, 现代密码学协议一般**混合使用对称加密和非对称加密** 以发挥各自的优势, 如使用非对称加密方案来安全地生成、交换对称加密所需的共享密钥, 然后使用对称加密方案来加密实际的消息内容.

1. KEM: 仅使用非对称加密算法加密另一个密钥, 实际数据的加解密由该密钥完成. 前面介绍 TLS 1.3 后量子密码学改进时提到过 ML-KEM, 这就是一个典型的 KEM.

   RSA-OAEP, RSA-KEM, ECIES-KEM 和 PSEC-KEM. 都是 KEM 加密方案.

2. IKS: 集成加密方案, 在 KEM 技术上添加 KDF 等其他密码学算法达成更高安全性, 此处不展开.

### RSA 公钥密码系统

RSA 密码系统是最早的公钥密码系统之一, 它基于模幂的数学和 RSA 问题的计算难度以及密切相关的整数分解问题 (IFP). RSA 算法以其作者的首字母命名, 并在计算机密码学的早期广泛使用.

RSA 密码系统是典型的公钥密码系统, 完整提供了密钥对生成、加解密、密钥交换和数字签名等功能.

RSA 私钥长度可以是 1024, 2048, 3072 或 4096 位及更长, 但 3072 位是目前的推荐最小长度, 过长的密钥也会导致加解密速度过慢.

#### 示例

2048 位 RSA 公钥示例 (表示为 2048 位十六进制整数模数 n 和 24 位公指数 e):

```
n = 0xa709e2f84ac0e21eb0caa018cf7f697f774e96f8115fc2359e9cf60b1dd8d4048d974cdf8422bef6be3c162b04b916f7ea2133f0e3e4e0eee164859bd9c1e0ef0357c142f4f633b4add4aab86c8f8895cd33fbf4e024d9a3ad6be6267570b4a72d2c34354e0139e74ada665a16a2611490debb8e131a6cffc7ef25e74240803dd71a4fcd953c988111b0aa9bbc4c57024fc5e8c4462ad9049c7f1abed859c63455fa6d58b5cc34a3d3206ff74b9e96c336dbacf0cdd18ed0c66796ce00ab07f36b24cbe3342523fd8215a8e77f89e86a08db911f237459388dee642dae7cb2644a03e71ed5c6fa5077cf4090fafa556048b536b879a88f628698f0c7b420c4b7
e = 0x010001
```

以 RSA PKCS#8 PEM ASN.1 格式编码:

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEApwni+ErA4h6wyqAYz39p
f3dOlvgRX8I1npz2Cx3Y1ASNl0zfhCK+9r48FisEuRb36iEz8OPk4O7hZIWb2cHg
7wNXwUL09jO0rdSquGyPiJXNM/v04CTZo61r5iZ1cLSnLSw0NU4BOedK2mZaFqJh
FJDeu44TGmz/x+8l50JAgD3XGk/NlTyYgRGwqpu8TFcCT8XoxEYq2QScfxq+2FnG
NFX6bVi1zDSj0yBv90uelsM226zwzdGO0MZnls4AqwfzayTL4zQlI/2CFajnf4no
agjbkR8jdFk4je5kLa58smRKA+ce1cb6UHfPQJD6+lVgSLU2uHmoj2KGmPDHtCDE
twIDAQAB
-----END PUBLIC KEY-----
```

2048 位 RSA 私钥示例, 对应于上述给定的公钥 (表示为十六进制 2048 位整数模数 n 和 2048 位秘密指数 d):

```
n = 0xa709e2f84ac0e21eb0caa018cf7f697f774e96f8115fc2359e9cf60b1dd8d4048d974cdf8422bef6be3c162b04b916f7ea2133f0e3e4e0eee164859bd9c1e0ef0357c142f4f633b4add4aab86c8f8895cd33fbf4e024d9a3ad6be6267570b4a72d2c34354e0139e74ada665a16a2611490debb8e131a6cffc7ef25e74240803dd71a4fcd953c988111b0aa9bbc4c57024fc5e8c4462ad9049c7f1abed859c63455fa6d58b5cc34a3d3206ff74b9e96c336dbacf0cdd18ed0c66796ce00ab07f36b24cbe3342523fd8215a8e77f89e86a08db911f237459388dee642dae7cb2644a03e71ed5c6fa5077cf4090fafa556048b536b879a88f628698f0c7b420c4b7
d = 0x10f22727e552e2c86ba06d7ed6de28326eef76d0128327cd64c5566368fdc1a9f740ad8dd221419a5550fc8c14b33fa9f058b9fa4044775aaf5c66a999a7da4d4fdb8141c25ee5294ea6a54331d045f25c9a5f7f47960acbae20fa27ab5669c80eaf235a1d0b1c22b8d750a191c0f0c9b3561aaa4934847101343920d84f24334d3af05fede0e355911c7db8b8de3bf435907c855c3d7eeede4f148df830b43dd360b43692239ac10e566f138fb4b30fb1af0603cfcf0cd8adf4349a0d0b93bf89804e7c2e24ca7615e51af66dccfdb71a1204e2107abbee4259f2cac917fafe3b029baf13c4dde7923c47ee3fec248390203a384b9eb773c154540c5196bce1
```

以传统的 RSA PKCS#1 PEM ASN.1 格式编码, 看起来更长一些:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEApwni+ErA4h6wyqAYz39pf3dOlvgRX8I1npz2Cx3Y1ASNl0zf
hCK+9r48FisEuRb36iEz8OPk4O7hZIWb2cHg7wNXwUL09jO0rdSquGyPiJXNM/v0
4CTZo61r5iZ1cLSnLSw0NU4BOedK2mZaFqJhFJDeu44TGmz/x+8l50JAgD3XGk/N
lTyYgRGwqpu8TFcCT8XoxEYq2QScfxq+2FnGNFX6bVi1zDSj0yBv90uelsM226zw
zdGO0MZnls4AqwfzayTL4zQlI/2CFajnf4noagjbkR8jdFk4je5kLa58smRKA+ce
1cb6UHfPQJD6+lVgSLU2uHmoj2KGmPDHtCDEtwIDAQABAoIBABDyJyflUuLIa6Bt
ftbeKDJu73bQEoMnzWTFVmNo/cGp90CtjdIhQZpVUPyMFLM/qfBYufpARHdar1xm
qZmn2k1P24FBwl7lKU6mpUMx0EXyXJpff0eWCsuuIPonq1ZpyA6vI1odCxwiuNdQ
oZHA8MmzVhqqSTSEcQE0OSDYTyQzTTrwX+3g41WRHH24uN479DWQfIVcPX7u3k8U
jfgwtD3TYLQ2kiOawQ5WbxOPtLMPsa8GA8/PDNit9DSaDQuTv4mATnwuJMp2FeUa
9m3M/bcaEgTiEHq77kJZ8srJF/r+OwKbrxPE3eeSPEfuP+wkg5AgOjhLnrdzwVRU
DFGWvOECgYEAyIk7F0S0AGn2aryhw9CihDfimigCxEmtIO5q7mnItCfeQwYPsX72
1fLpJNgfPc9DDfhAZ2hLSsBlAPLUOa0Cuny9PCBWVuxi1WjLVaeZCV2bF11mAgW2
fjLkAXT34IX+HZl60VoetSWq9ibfkJHeCAPnh/yjdB3Vs+2wxNkU8m8CgYEA1Tzm
mjJq7M6f+zMo7DpRwFazGMmrLKFmHiGBY6sEg7EmoeH2CkAQePIGQw/Rk16gWJR6
DtUZ9666sjCH6/79rx2xg+9AB76XTFFzIxOk9cm49cIosDMk4mogSfK0Zg8nVbyW
5nEb//9JCrZ18g4lD3IrT5VJoF4MhfdBUjAS1jkCgYB+RDIpv3+bNx0KLgWpFwgN
Omb667B6SW2ya4x227KdBPFkwD9HYosnQZDdOxvIvmUZObPLqJan1aaDR2Krgi1S
oNJCNpZGmwbMGvTU1Pd+Nys9NfjR0ykKIx7/b9fXzman2ojDovvs0W/pF6bzD3V/
FH5HWKLOrS5u4X3JJGqVDwKBgQCd953FwW/gujld+EpqpdGGMTRAOrXqPC7QR3X5
Beo0PPonlqOUeF07m9/zsjZJfCJBPM0nS8sO54w7ESTAOYhpQBAPcx/2HMUsrnIj
HBxqUOQKe6l0zo6WhJQi8/+cU8GKDEmlsUlS3iWYIA9EICJoTOW08R04BjQ00jS7
1A1AUQKBgHlHrV/6S/4hjvMp+30hX5DpZviUDiwcGOGasmIYXAgwXepJUq0xN6aa
lnT+ykLGSMMY/LABQiNZALZQtwK35KTshnThK6zB4e9p8JUCVrFpssJ2NCrMY3SU
qw87K1W6engeDrmunkJ/PmvSDLYeGiYWmEKQbLQchTxx1IEddXkK
-----END RSA PRIVATE KEY-----
```

#### RSA 密钥生成

略, 相关应用参见加密库的文档.

#### RSA 加解密

RSA 一次只能加密/解密一个(大)整数, 一般采用 OAEP 对消息编码为一个个整数再逐个加解密.

#### RSA 签名与验证

参见后文介绍数字签名算法.

### ECC 公钥密码系统

ECC 椭圆曲线密码学于 1985 年被首次提出, 并于 2004 年开始被广泛应用. ECC 被认为是 RSA 的继任者, 新一代的非对称加密算法. 其最大的特点在于相同密码强度下, ECC 的密钥和签名的大小都要显著低于 RSA. 256 位的 ECC 密钥, 安全性与 3072 位的 RSA 密钥安全性相当. ECC 的密钥对生成、密钥交换与签名算法的速度都要比 RSA 快.

ECC 中的椭圆曲线可以以多种形式呈现, 这些形式被证明是**同构**的:

- 魏尔施特拉斯 (Weierstrass) 形式: 这是最常见的椭圆曲线形式, 其方程为 $y^2=x^3+ax+b$

  如: $y^2=x^3+7$ (secp256k1, 加密货币领域常用).

  > 需要指出, NSA 推荐的 secp256r1 (NIST P-256, prime256v1, nist256p1) 和 secp256k1 名称只有一字之差, k 和 r 的不同. 其中 k 表示 Koblitz (ECC 发明人), 而 r 表示随机, 即参数是随机选取的, 但美国国家安全局 NSA 并没有公布随机数的挑选规则, 有质疑称 NSA 可能对随机数动过手脚, 让破解难度大幅降低.

- 蒙哥马利 (Montgomery) 形式: 其方程为 $By^2=x^3+Ax^2+x$

  如 Curve25519: $y^2 = x^3+486662x^2+x$.

- 爱德华兹 (Edwards) 形式: 其方程为 $x^2+y^2=1+dx^2y^2$

  如 Ed448: $x^2+y^2=1-39081x^2y^2$. (也可以写成蒙哥马利形式, 称 Curve448).
  
  一般而言, Edwards 形式性能更好, 如 Ed25519 是基于 Curve25519 同构地 "扭曲" 而来, 专为计算速度优化而设计的.

我们推荐使用 Curve52219 / Ed25519 以及 Curve448 / Ed448, 其性能和安全性均优于 NSA 推荐的 NIST P-256 (secp256r1), NIST P-384 (secp384r1), NIST P-521 (secp521r1) 等曲线. 参见 <https://safecurves.cr.yp.to>.

> 轶事一则
>
> 参考: <https://www.cnblogs.com/greencollar/p/14363535.html>
>
> Daniel J. Bernstein 是世界著名的密码学家, 他在大学曾经开设过一门 UNIX 系统安全的课程给学生, 结果一学期下来, 发现了 UNIX 程序中的 91 个安全漏洞; 他早年在美国依然禁止出口加密算法时, 曾因为把自己设计的加密算法发布到网上遭到了美国政府的起诉, 他本人抗争六年, 最后美国政府撤销所有指控, 目前另一个非常火的高性能安全流密码 ChaCha20 也是出自 Bernstein 之手.
>
> Curve25519 自 2006 年发表以来, 除了学术界无人问津. 但自 2013 年斯诺登曝光棱镜计划后, 该算法突然大火. Curve25519 安全性高, 不同于 NIST 推荐的 P-256 等曲线, Curve25519 的参数是公开透明的, 任何人都可以验证其安全性, 而 NSA 推荐的 NIST P-256 等曲线方程的系数使用了来历不明的随机种子, 在密码学上难以验证其安全性; 此外, Curve25519 的设计也充分考虑了缓存攻击等, 尽可能避免分支跳转, 在实践上安全性也非常高.

## 数字签名算法 (digital signature algorithm, DSA)

在密码学中, 数字签名为数字文档提供消息身份验证、完整性和不可否认性. 数字签名基于公钥密码系统, 消息签名由私有密钥执行, 消息验证由相应的公钥执行.

数字签名的过程本质还是加解密, 但是加解密的对象变成了消息的哈希值: 使用私钥签名 (加密哈希值), 公钥验证 (解密后比对实际的哈希值).

### RSA 数字签名算法

即使用 RSA 公钥密码系统的数字签名算法.

> 刚刚好, 哈希值就是一个大整数.

### ECC 数字签名算法

即使用 ECC 公钥密码系统的数字签名算法. ECC 相对于 RSA, 在相同的安全强度下, 使用更短的密钥长度, 因此在计算和存储方面更为高效.

常用的 ECC 数字签名算法包括:

1. ECDSA: 在经典 Weierstrass 形式的有限域上使用加密椭圆曲线.

   > 出于特定实体影响, 目前的 ECC 证书一般使用 ECDSA 数字签名算法.
   >
   > 需要指出, DSA 是 digital signature algorithm (数字签名算法) 的缩写, 但也可能特指 FIPS 186 Digital Signature Standard (DSS) 中定义的数字签名算法. 一般地, ECDSA 特指使用了 NIST P-256, NIST P-384 或 NIST P-521 曲线的 NIST DSA 的椭圆曲线实现.
1. EdDSA: 在 Edwards 形式的有限域上使用加密椭圆曲线.

   常用变体包括 Ed25519 (使用 Ed25519 曲线, SHA512 哈希算法) 或 Ed448 (使用 Ed448 曲线, SHAKE256 哈希算法).

我们推荐使用 EdDSA 作为数字签名算法, 原因前面已经提到过.

## 数字证书

(本节内容大部分摘录自 <https://thiscute.world/posts/about-tls-cert>) 的总结, 向原作者表示感谢!)

我们已经熟悉了对称密码与非对称密码两个密码学体系:

- 对称密码算法: 计算速度快、安全强度高, 但是缺乏安全保存、管理乃至交换密钥的手段 (PSK 一旦泄露, 则所有使用该密钥加密的通信均不再安全).
- 非对称密码算法: 计算速度慢, 但是它给出了安全的密钥交换算法 DHE/ECDHE, 且公钥是可公开的, 这降低了密钥的保存与管理难度.

但是问题来了:

- 公钥该如何分发? 比如 Alice 跟 Bob 交换公钥时, 如何确定收到的确实是对方的公钥, 也就是说如何确认公钥的真实性、完整性、认证其来源身份? (DH 是匿名的无认证的, 无法防止中间人攻击)
- 如果 Alice 的私钥泄漏了, 她该如何作废自己旧的公钥?

数字证书与公钥基础架构就是为了解决上述问题而设计的.

首先简单介绍下 PKI (Public Key Infrastructure, 公钥基础架构), 它是一组由硬件、软件、参与者、管理政策与流程组成的基础架构, 其目的在于创造、管理、分配、使用、存储以及撤销数字证书. PKI 是一个总称, 而并非指单独的某一个规范或标准, 因此显然数字证书的规范 (X.509)、存储格式等都是 PKI 的一部分.

### 公钥证书

前面我们介绍了公钥密码系统存在的一个问题是 "在分发公钥时, 难以确认公钥的真实性、完整性及其来源身份". PKI 通过数字证书和证书认证机构来解决这个问题.

数字证书指的其实就是公钥证书 (也可直接简称为证书). 在现代网络通讯中通行的公钥证书格式标准名为 X.509 v3, 由 [RFC5280] 定义, 被广泛应用在 TLS 等众多加密通讯协议中.

一个公钥证书内容应当包括:

- 证书
  - 序列号 (Serial Number): 用以识别每一张证书, 在作废证书的时候会用到它.
  - 版本: 证书的规范版本.
  - 公钥 (Public Key): 证书所包含的公钥.
  - 公钥指纹: 即公钥的 Hash 值, 当前大部分证书都使用 SHA256 计算此指纹.
  - 公钥用途 (Key Usage + Extended Key Usage): 记录了此证书可用于哪些用途, 如数字签名、身份认证等.
  - 主体 (Subject): 即姓名、组织、邮箱、地址等证书拥有者的个人信息.
  - 证书有效期的开始时间、结束时间 (Not Before + Not After): 为了确保安全性, 每个证书都会记录一个自身的有效期.
  - 签发者 (Issuer): 签发此证书的实体.
  - 其他拓展信息.
- 数字签名 (Signature): 对整个证书计算数字签名, 来确保这些数据的真实性、完整性, 确保证书未被恶意篡改/伪造.

  此数字签名由证书签发者使用其私钥+证书内容计算得出.
- 数字签名算法 (Signature Algorithm): 证书所使用的签名算法, 常用的有 RSA-SHA-256 与 ECDSA-SHA-256.

每个证书都有唯一的 ID, 这样在私钥泄漏的情况下, 我们可以通过公钥基础设施的 OCSP (Online Certificate Status Protocol) 协议吊销某个证书. 吊销证书的操作还是比较罕见的, 毕竟私钥泄漏并不容易遇到, 因此这里就略过不提了, 有需要的可以自行搜索.

感兴趣的, 可以使用浏览器查看证书详情的功能查看一下自己常用网站的证书, 了解一下证书的具体内容, 这里是旧版 Firefox 查看谷歌网站的示例:

<div align="center">
  <img src="./google-cert-content.webp" alt="Google's Cert" style="max-width: 50%;">
</div>

### 证书签发机构 (CA) 与证书链

前面介绍证书内容时, 提到了签发者会使用签发者私钥和证书内容生成数字签名, 显然需要使用签发者公钥来验证被签发证书中的签名. 但是问题又来了, 签发者的公钥该如何分发? 这不又回到了最初的问题吗?

为此, PKI 引入了一个可信第三方 (Trusted third party, TTP) 来解决这个问题.

在 Alice 与 Bob 的案例中, 就是说还有个可信第三方 Eve, 他使用自己的私钥为自己的公钥证书签了名, 生成了一个*自签名*证书, 并且已经提前将这个自签名证书分发(比如当面交付、预置在操作系统中等手段)给了 Alice 跟 Bob.

- 现在 Alice 首先使用自己的公钥以及个人信息制作了自己的公钥证书, 但是这个证书还缺乏一个 Issuer 以及数字签名.
- Alice 找到 Eve, 将这个 CSR 文件提交给 Eve, 希望 Eve 为自己的证书签名. 此时, 我们称之为提交**CSR** (Certificate Signing Request, CSR)
- Eve 验证了 Alice 的身份后, 再使用这个 CSR 签发出完整的证书文件 (Issuer 就填 Eve, 然后 Eve 使用自己的私钥计算出证书的数字签名, 这样就得到了最终的证书), 交付给 Alice.

  Eve 曾经 "跨越千里之遥", 将自己的公钥证书分发给了 Bob, 所以在给 Alice 签发证书时, 他显然可能会要求 Alice 给付*签名费*. 目前许多证书机构就是靠这个赚钱的, 当然也有非盈利的证书颁发机构, 如 ZeroSSL 等 (真的是做慈善么).
- 现在 Alice 再将经 Eve 签名的证书发送给 Bob
- Bob 收到证书后, 看到 Issuer 是 Eve, 于是找出以前 Eve 给他的*自签名*证书, 然后使用其中的公钥验证收到的证书.

  如果验证成功, 就说明证书的内容是经过 Eve 认证的. 如果 Eve 没老糊涂了, 那这个证书应该确实就是 Alice 的. 否则, 这是某个攻击者伪造的证书.

在现实世界中, Eve 这个角色被称作 CA (Certification Authority, 证书认证机构), 全世界只有几十家这样的权威机构, 它们*都通过了各大软件厂商的严格审核*, 从而将根证书（CA 证书）直接内置于主流操作系统与浏览器中, 也就是说早就提前分发给了因特网世界的几乎所有用户. 由于许多操作系统或软件的更新迭代缓慢, 根证书的有效期通常都在十年以上.

但是, 如果 CA 机构直接使用自己的私钥处理各种证书签名请求, 这将是非常危险的. 因为全世界有海量的 HTTPS 网站, 也就是说有海量的证书需求, 可一共才几十家 CA 机构. 频繁的动用私钥会产生私钥泄漏的风险, 如果这个私钥泄漏了, 那将直接影响海量网站的安全性.

PKI 架构使用数字证书链的机制来解决这个问题:

<div align="center">
  <img src="./chain-of-trust.webp" alt="Chain of trust" style="max-width: 50%;">
</div>

- CA 机构首先生成自己的根证书与私钥, 并使用私钥给根证书签名

  因为私钥跟证书本身就是一对, 因此根证书也被称作自签名证书.
- CA 交付根证书给各大软硬件厂商, 内置在主流的操作系统与浏览器中.
- CA 使用私钥签发一些所谓的中间证书, 之后就把私钥雪藏了, 非必要不会再拿出来使用.

  根证书的私钥通常离线存储在安全地点; 中间证书的有效期通常会比根证书短一些; 部分中间证书会被作为备份使用, 平常不会启用.

  - CA 机构使用这些中间证书的私钥, 为用户提交的所有 CSR 请求签名.
  - CA 机构也可以使用中间证书的私钥, 为子一级 CA 机构签发中间证书, 形成更长的证书链.
  - 不同 CA 机构也可能执行交叉签名, 共同为对方的中间证书签名. 客户端只要安装有其中任何一个根证书, 就能验证该中间证书.

上述这个全球互联网上, 由证书认证机构、操作系统与浏览器内置的根证书、TLS 加密认证协议、OCSP 证书吊销协议等等组成的架构, 我们可以称它为 **Web PKI**.

Web PKI 通常是*可信*的, 但是并不意味着它们*可靠*.

> 可以说, 谁*掌握*了 CA, 谁就能对如今互联网安全所仰赖的 TLS 通信执行 MitM.

在更高安全性要求的场景中, 搭建私有 PKI 架构是有必要的, 如支付场景下银行 APP 等要求安装特定数字证书, 即安装了一个由支付机构控制的私有 CA 证书, 或者以 U 盾形式存在的硬件介质的证书; 又或者内部通讯系统, 也会使用私有 CA 证书来签发内部服务器的证书. 这种情况下, 私有 CA 证书需要被提前手动安装在客户端设备上, 否则会提示不受信任的证书错误.

### TLS 证书主体 (Subject)

TLS 证书的主体字段包含了证书持有者的身份信息, 我们主要关注其中的通用名称 (Common Name, CN) 和主体备用名称 (Subject Alternative Name, SAN).

CN 是证书的 "主域名", SAN 则是该证书所覆盖的所有域名或 IP 地址. 由于 CN 字段只能有一个, 无法满足单证书保护多个子域的需求, 实际上已被弃用, 我们通过 SAN 字段来指定证书所覆盖的域名/IP 地址.

一般地, 我们使用的证书均为域名证书, 但也有 IP 证书 (只能是公网 IP, 难以自动化申请, 而且收费一般不便宜, 目前应该只有 ZeroSSL 提供了*免费*的 IP 证书), 如著名的 `https://1.1.1.1`.

SAN 支持配置泛域名, 即 `*.example.com` 的形式, 但不支持多级泛域名, 即 `*.*.example.com` 的形式; `*.example.com` 只能匹配到同级子域名 `a.example.com`, 不能匹配 `a.b.example.com` 等.

SAN 可以设置很多个, 如:

<div align="center">
  <img src="./1.1.1.1-cert.webp" alt="1.1.1.1 Cert" style="max-width: 50%;">
</div>

## 补充内容: 证书以及公私钥的(文本)编码

由于历史原因, 实践上对证书以及公私钥的存储格式可谓五花八门, 这里参考了 RFC 7468 以及 <https://thiscute.world/posts/about-tls-cert/#pki>, 试图做一个实用意义上比较全面的总结.

### 基本概念

1. X.509: PKI 标准, 规定了证书应该包含哪些信息 (但是未定义证书该如何存储). PKIX 即 "Public Key Infrastructure X.509".
1. ASN.1 (Abstract Syntax Notation One, 抽象语法标记法第 1 部分): 一种用于描述数据结构的标准, 广泛应用于通信协议和加密标准中. ASN.1 定义了数据的语法和语义, 使得不同系统之间能够理解和交换数据. 被应用于 X.509 证书、PKCS 标准等.

   感兴趣的, 可以参考 <https://letsencrypt.org/zh-cn/docs/a-warm-welcome-to-asn1-and-der/> 的总结.
1. DER (Distinguished Encoding Rules, 可辨别编码规则): ASN.1 的一种序列化规则, 用于将 ASN.1 数据结构编码为二进制格式. 定义于 X.690 标准中.

   DER 编码的证书*通常*使用 `.cer` 或 `.der` 作为文件扩展名, 但也有使用 `.crt` 的 (RFC 7468 要求使用此拓展名保存文本格式), 更有使用 `.cer` 保存 PEM 格式的证书的. 在实践上, 尝试以纯文本形式打开看看就知道了.
1. PEM (Privacy-Enhanced Mail, 隐私增强邮件): "隐私增强邮件" 自身已鲜少使用, 但其定义的基于文本的编码格式, 如今被广泛应用于证书(链)和公私钥的文本编码中. PEM 格式可以理解为对 DER 编码的证书或密钥进行 Base64 编码并添加封装边界.
1. PKCS (Public Key Cryptography Standards, 公钥密码学标准).

### PEM 格式

(以下来自 [RFC7468], 但是由于历史因素, 实践中 PEM 格式的使用也存在不完全符合 RFC 7468 的定义的情况)

文本编码以一行包含 "`-----BEGIN `"、标签和 "`-----`" 开始, 并以一行包含 "`-----END `"、标签和 "`-----`" 结束. 在这些行 (或称"封装边界") 之间, 是根据 RFC4648 第 4 节进行 Base64 编码的数据 (PEM [RFC1421] 称这部分数据为 "封装文本部分"). 封装边界之前允许存在数据, 解析器在处理此类数据时**绝不能**出错. 此外, 解析器**必须**忽略空白字符及其他非 Base64 字符, 且**必须**支持不同的换行规范.

编码数据的类型由 "`-----BEGIN `" (前置封装边界) 行中的类型标签决定. 例如, "`-----BEGIN CERTIFICATE-----`" 表示内容是一个 PKIX 证书. 生成器**必须**在 "`-----END `" (后置封装边界) 行使用与对应 "`-----BEGIN `" 行相同的标签. 标签由区分大小写的、全大写的零个或多个字符组成; 不得包含连续空格或连字符, 首尾也不得出现空格或连字符. 若标签不匹配, 解析器**可以**选择忽略后置封装边界中的标签而非报错: 现存实现中部分要求标签匹配, 部分不作要求.

"BEGIN" 或 "END" 与标签之间必须严格用一个空格字符 (SP) 分隔. 封装边界两端的连字符 (即短横线 "-") 必须正好五个, 不能多也不能少.

标签类型暗示编码数据遵循特定语法. 解析器**必须**能够优雅处理不符合规范的数据. 但需注意本文档发布前的解析器或生成器行为并不统一. 合法的解析器**可能**将内容解释为其他标签类型, 但应注意安全考量章节讨论的潜在风险. 本文档描述的标签标识的容器格式不与特定加密算法绑定, 这符合算法敏捷性原则. 这些格式采用 [RFC5280] 第 4.1.1.2 节所述的 ASN.1 AlgorithmIdentifier 结构.

与传统 PEM 编码 [RFC1421]、OpenPGP ASCII armor 及 OpenSSH 密钥文件格式不同, 文本编码*不*定义也不允许在数据旁编码头信息. 前置封装边界与 Base64 数据之间允许存在空白, 但生成器**绝不能**产生此类空白 (保留此空白区域是对 PEM "封装头部分" 定义的沿袭).

### PKCS #1 标准

专用于编码 RSA 公私钥, PEM 标签为 "RSA PUBLIC KEY" 和 "RSA PRIVATE KEY".

示例:

```
-----BEGIN RSA PUBLIC KEY-----
MIIBCgKCAQEApwni+ErA4h6wyqAYz39pf3dOlvgRX8I1npz2Cx3Y1ASNl0zfhCK+
9r48FisEuRb36iEz8OPk4O7hZIWb2cHg7wNXwUL09jO0rdSquGyPiJXNM/v04CTZ
o61r5iZ1cLSnLSw0NU4BOedK2mZaFqJhFJDeu44TGmz/x+8l50JAgD3XGk/NlTyY
gRGwqpu8TFcCT8XoxEYq2QScfxq+2FnGNFX6bVi1zDSj0yBv90uelsM226zwzdGO
0MZnls4AqwfzayTL4zQlI/2CFajnf4noagjbkR8jdFk4je5kLa58smRKA+ce1cb6
UHfPQJD6+lVgSLU2uHmoj2KGmPDHtCDEtwIDAQAB
-----END RSA PUBLIC KEY-----
```

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEApwni+ErA4h6wyqAYz39pf3dOlvgRX8I1npz2Cx3Y1ASNl0zf
hCK+9r48FisEuRb36iEz8OPk4O7hZIWb2cHg7wNXwUL09jO0rdSquGyPiJXNM/v0
4CTZo61r5iZ1cLSnLSw0NU4BOedK2mZaFqJhFJDeu44TGmz/x+8l50JAgD3XGk/N
lTyYgRGwqpu8TFcCT8XoxEYq2QScfxq+2FnGNFX6bVi1zDSj0yBv90uelsM226zw
zdGO0MZnls4AqwfzayTL4zQlI/2CFajnf4noagjbkR8jdFk4je5kLa58smRKA+ce
1cb6UHfPQJD6+lVgSLU2uHmoj2KGmPDHtCDEtwIDAQABAoIBABDyJyflUuLIa6Bt
ftbeKDJu73bQEoMnzWTFVmNo/cGp90CtjdIhQZpVUPyMFLM/qfBYufpARHdar1xm
qZmn2k1P24FBwl7lKU6mpUMx0EXyXJpff0eWCsuuIPonq1ZpyA6vI1odCxwiuNdQ
oZHA8MmzVhqqSTSEcQE0OSDYTyQzTTrwX+3g41WRHH24uN479DWQfIVcPX7u3k8U
jfgwtD3TYLQ2kiOawQ5WbxOPtLMPsa8GA8/PDNit9DSaDQuTv4mATnwuJMp2FeUa
9m3M/bcaEgTiEHq77kJZ8srJF/r+OwKbrxPE3eeSPEfuP+wkg5AgOjhLnrdzwVRU
DFGWvOECgYEAyIk7F0S0AGn2aryhw9CihDfimigCxEmtIO5q7mnItCfeQwYPsX72
1fLpJNgfPc9DDfhAZ2hLSsBlAPLUOa0Cuny9PCBWVuxi1WjLVaeZCV2bF11mAgW2
fjLkAXT34IX+HZl60VoetSWq9ibfkJHeCAPnh/yjdB3Vs+2wxNkU8m8CgYEA1Tzm
mjJq7M6f+zMo7DpRwFazGMmrLKFmHiGBY6sEg7EmoeH2CkAQePIGQw/Rk16gWJR6
DtUZ9666sjCH6/79rx2xg+9AB76XTFFzIxOk9cm49cIosDMk4mogSfK0Zg8nVbyW
5nEb//9JCrZ18g4lD3IrT5VJoF4MhfdBUjAS1jkCgYB+RDIpv3+bNx0KLgWpFwgN
Omb667B6SW2ya4x227KdBPFkwD9HYosnQZDdOxvIvmUZObPLqJan1aaDR2Krgi1S
oNJCNpZGmwbMGvTU1Pd+Nys9NfjR0ykKIx7/b9fXzman2ojDovvs0W/pF6bzD3V/
FH5HWKLOrS5u4X3JJGqVDwKBgQCd953FwW/gujld+EpqpdGGMTRAOrXqPC7QR3X5
Beo0PPonlqOUeF07m9/zsjZJfCJBPM0nS8sO54w7ESTAOYhpQBAPcx/2HMUsrnIj
HBxqUOQKe6l0zo6WhJQi8/+cU8GKDEmlsUlS3iWYIA9EICJoTOW08R04BjQ00jS7
1A1AUQKBgHlHrV/6S/4hjvMp+30hX5DpZviUDiwcGOGasmIYXAgwXepJUq0xN6aa
lnT+ykLGSMMY/LABQiNZALZQtwK35KTshnThK6zB4e9p8JUCVrFpssJ2NCrMY3SU
qw87K1W6engeDrmunkJ/PmvSDLYeGiYWmEKQbLQchTxx1IEddXkK
-----END RSA PRIVATE KEY-----
```

如今已经统一转向使用 PKCS #8 格式保存私钥, 公钥则按 [RFC7468] Section 13 所述的格式保存.

### ANSI X9.62 (ITU-T X.894) 标准

类似 PKCS #1 标准, 但此标准仅用于编码 ECC 私钥, PEM 标签为 "EC PRIVATE KEY".

示例 (使用 `openssl ecparam -name secp384r1 -genkey -noout -out privkey.pem` 生成):

```
-----BEGIN EC PRIVATE KEY-----
MIGkAgEBBDDhhKJcKC7jnRbKLYKAzY1f/9aC9JGGuqqo2d4FejHSi/TOZiLf3JhP
xAb+IJdQYEqgBwYFK4EEACKhZANiAARPyjgUZlw+IWfP/+PD8m1HZRWbEvUB+ai9
pTV63SI7MWw5OzM3qKwTUNP+NK3xFt14A5lJwFMzl5AB7QhJYarW5ra3e2yhP8Vn
oC9E7Om31FLwVX+AHTM4Rg0ZQQoqH0c=
-----END EC PRIVATE KEY-----
```

类似地, 如今已经统一转向使用 PKCS #8 格式保存私钥.

### PKCS #7 / CMS 标准

> PKCS #7 / CMS, 是一个多用途的证书描述格式. 它包含一个数据填充规则, 这个填充规则常被用在需要数据填充的分组加密、数字签名等算法中.
>
> 由于历史原因, 有使用 PKCS #7 标准保存证书的, 后缀通常使用 `.p7b` 或者 `.p7c`, 可以使用 DER 或 PEM 格式保存:
>
> <div align="center">
>  <img src="./p7b.png" alt=".p7b in real world" style="max-width: 50%;">
> </div>
>
> 以下内容摘抄自 [RFC7468] Section 8 和 Section 9.

PKCS #7 加密消息语法结构使用 "PKCS7" 标签进行编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 ContentInfo 结构, 如 [RFC2315] 所述.

```
-----BEGIN PKCS7-----
MIHjBgsqhkiG9w0BCRABF6CB0zCB0AIBADFho18CAQCgGwYJKoZIhvcNAQUMMA4E
CLfrI6dr0gUWAgITiDAjBgsqhkiG9w0BCRADCTAUBggqhkiG9w0DBwQIZpECRWtz
u5kEGDCjerXY8odQ7EEEromZJvAurk/j81IrozBSBgkqhkiG9w0BBwEwMwYLKoZI
hvcNAQkQAw8wJDAUBggqhkiG9w0DBwQI0tCBcU09nxEwDAYIKwYBBQUIAQIFAIAQ
OsYGYUFdAH0RNc1p4VbKEAQUM2Xo8PMHBoYdqEcsbTodlCFAZH4=
-----END PKCS7-----
```

标签 "CERTIFICATE CHAIN" 曾用于表示仅包含证书列表的退化 (degenerated) PKCS #7 结构 (参见 [RFC2315] 第 9 节). 多个现代工具已不再支持此标签. 生成器**绝不能**生成 "CERTIFICATE CHAIN" 标签. 解析器**绝不能**将 "CERTIFICATE CHAIN" 视为等同于 "PKCS7". PKCS #7 是一个已被 CMS [RFC5652] 长期取代的旧规范. 在 CMS 可作为替代方案时, 实现**绝不能**生成 PKCS #7.

CMS 结构使用 "CMS" 标签进行编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 ContentInfo 结构, 如 [RFC5652] 所述.

```
-----BEGIN CMS-----
MIGDBgsqhkiG9w0BCRABCaB0MHICAQAwDQYLKoZIhvcNAQkQAwgwXgYJKoZIhvcN
AQcBoFEET3icc87PK0nNK9ENqSxItVIoSa0o0S/ISczMs1ZIzkgsKk4tsQ0N1nUM
dvb05OXi5XLPLEtViMwvLVLwSE0sKlFIVHAqSk3MBkkBAJv0Fx0=
-----END CMS-----
```

CMS 是 IETF 对 PKCS #7 的继承. [RFC5652] 第1.1.1节描述了自 PKCS #7 v1.5 以来的变更. 实现方案在可选时**必须**生成 CMS 格式, 以促进互操作性和前向兼容性.

### PKCS #8 标准

> 此标准专用于编码私钥, 且允许对私钥加密, 是现代非对称密钥存储的事实标准.
>
> 私钥建议使用 `.key` 作为文件扩展名, 但也有相当一部分人使用通用拓展名 `.pem` 的.
>
> 可以使用 `openssl pkcs8 -topk8 -inform PEM -in private-key.pem -outform PEM -nocrypt -out private-key-pkcs8.pem` 来将传统私钥转换为 PKCS #8 格式.
>
> 以下内容摘抄自 [RFC7468] Section 10 和 Section 11.

未加密的 PKCS #8 私钥信息语法结构 (PrivateKeyInfo, 现更名为非对称密钥包 OneAsymmetricKey) 使用 "PRIVATE KEY" 标签编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 PrivateKeyInfo 结构, 如 PKCS #8 [RFC5208] 所述, 或 OneAsymmetricKey 结构, 如 [RFC5958] 所述. 两者语义相同, 可通过版本号区分.

```
-----BEGIN PRIVATE KEY-----
MIGHTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQglQanBRiYVPX7F2Rd
4CqyjEN0K4qfHw4tM/yMIh21wamhRANCAARsxaI4jT1b8zbDlFziuLngPcExbYzz
ePAHUmgWL/ZCeqlODF/l/XvimkjaWC2huu1OSWB9EKuG+mKFY2Y5k+vF
-----END PRIVATE KEY-----
```

加密的 PKCS #8 私钥信息语法结构 (EncryptedPrivateKeyInfo, [RFC5958] 中名称相同) 使用 "ENCRYPTED PRIVATE KEY" 标签编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 PrivateKeyInfo 结构, 如 PKCS #8 [RFC5208] 和 [RFC5958] 所述.

```
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIHNMEAGCSqGSIb3DQEFDTAzMBsGCSqGSIb3DQEFDDAOBAghhICA6T/51QICCAAw
FAYIKoZIhvcNAwcECBCxDgvI59i9BIGIY3CAqlMNBgaSI5QiiWVNJ3IpfLnEiEsW
Z0JIoHyRmKK/+cr9QPLnzxImm0TR9s4JrG3CilzTWvb0jIvbG3hu0zyFPraoMkap
8eRzWsIvC5SVel+CSjoS2mVS87cyjlD+txrmrXOVYDE+eTgMLbrLmsWh3QkCTRtF
QC7k0NNzUHTV9yGDwfqMbw==
-----END ENCRYPTED PRIVATE KEY-----
```

### PKCS #12 标准

PKCS #12 是一个归档文件格式, 不是文本编码, 但这里也顺带说一下了. 主要用于存储多个私钥及相关的 X.509 证书, 常用拓展名为 `.p12`, `.pfx`. 因为保存了私钥, 为了安全性它通常是加密的, 需要解密后才能使用.

安卓开发者应该比较熟悉, APK 签名证书通常就是使用 PKCS #12 格式存储的, 拓展名为 `.keystore` 或者 `.jks` 而已.

### 公钥 (Public Key) PEM 编码标准 (RFC 7468 Section 13)

公钥信息结构 (SubjectPublicKeyInfo) 使用 "PUBLIC KEY" 标签进行编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 SubjectPublicKeyInfo 结构, 如 [RFC5280] 第 4.1.2.7 节所述.

```
-----BEGIN PUBLIC KEY-----
MHYwEAYHKoZIzj0CAQYFK4EEACIDYgAEn1LlwLN/KBYQRVH6HfIMTzfEqJOVztLe
kLchp2hi78cCaMY81FBlYs8J9l7krc+M4aBeCGYFjba+hiXttJWPL7ydlE+5UG4U
Nkn3Eos8EiZByi9DVsyfy9eejh+8AXgp
-----END PUBLIC KEY-----
```

### 证书 (Certificate) PEM 编码标准 (RFC 7468 Section 5)

> 这里的 "证书", 指互联网 X.509 公钥基础设施证书及证书吊销列表 (Certificate Revocation Lists, CRL) 规范 [RFC5280] 中的 "证书".
>
> 以下内容摘抄自 [RFC7468] Section 5.

5.1. 编码

公钥证书使用 "CERTIFICATE" 标签进行编码.

编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 Certificate 结构, 如 [RFC5280] 第 4 节所述.

```
-----BEGIN CERTIFICATE-----
MIICLDCCAdKgAwIBAgIBADAKBggqhkjOPQQDAjB9MQswCQYDVQQGEwJCRTEPMA0G
A1UEChMGR251VExTMSUwIwYDVQQLExxHbnVUTFMgY2VydGlmaWNhdGUgYXV0aG9y
aXR5MQ8wDQYDVQQIEwZMZXV2ZW4xJTAjBgNVBAMTHEdudVRMUyBjZXJ0aWZpY2F0
ZSBhdXRob3JpdHkwHhcNMTEwNTIzMjAzODIxWhcNMTIxMjIyMDc0MTUxWjB9MQsw
CQYDVQQGEwJCRTEPMA0GA1UEChMGR251VExTMSUwIwYDVQQLExxHbnVUTFMgY2Vy
dGlmaWNhdGUgYXV0aG9yaXR5MQ8wDQYDVQQIEwZMZXV2ZW4xJTAjBgNVBAMTHEdu
dVRMUyBjZXJ0aWZpY2F0ZSBhdXRob3JpdHkwWTATBgcqhkjOPQIBBggqhkjOPQMB
BwNCAARS2I0jiuNn14Y2sSALCX3IybqiIJUvxUpj+oNfzngvj/Niyv2394BWnW4X
uQ4RTEiywK87WRcWMGgJB5kX/t2no0MwQTAPBgNVHRMBAf8EBTADAQH/MA8GA1Ud
DwEB/wQFAwMHBgAwHQYDVR0OBBYEFPC0gf6YEr+1KLlkQAPLzB9mTigDMAoGCCqG
SM49BAMCA0gAMEUCIDGuwD1KPyG+hRf88MeyMQcqOFZD0TbVleF+UsAGQ4enAiEA
l4wOuDwKQa+upc8GftXE2C//4mKANBC6It01gUaTIpo=
-----END CERTIFICATE-----
```

以上为示例.

历史上曾使用过 "X509 CERTIFICATE" 标签, 以及较少使用的 "X.509 CERTIFICATE" 标签. 符合本文件的生成器**必须**生成 "CERTIFICATE" 标签, **绝不能**生成 "X509 CERTIFICATE" 或 "X.509 CERTIFICATE" 标签. 解析器**绝不能**将 "X509 CERTIFICATE" 或 "X.509 CERTIFICATE" 视为等同于 "CERTIFICATE", 但有效的例外可能是为了向后兼容性 (可能附带警告).

5.2. 说明性文本

已知许多工具在 PKIX 证书的 BEGIN 行之前和 END 行之后会输出说明性文本, 比其他类型的证书更为常见. 如果输出此类文本, 应使其与证书相关, 例如提供证书中关键数据元素的文本表示.

```
Subject: CN=Atlantis
Issuer: CN=Atlantis
Validity: from 7/9/2012 3:10:38 AM UTC to 7/9/2013 3:10:37 AM UTC
-----BEGIN CERTIFICATE-----
MIIBmTCCAUegAwIBAgIBKjAJBgUrDgMCHQUAMBMxETAPBgNVBAMTCEF0bGFudGlz
MB4XDTEyMDcwOTAzMTAzOFoXDTEzMDcwOTAzMTAzN1owEzERMA8GA1UEAxMIQXRs
YW50aXMwXDANBgkqhkiG9w0BAQEFAANLADBIAkEAu+BXo+miabDIHHx+yquqzqNh
Ryn/XtkJIIHVcYtHvIX+S1x5ErgMoHehycpoxbErZmVR4GCq1S2diNmRFZCRtQID
AQABo4GJMIGGMAwGA1UdEwEB/wQCMAAwIAYDVR0EAQH/BBYwFDAOMAwGCisGAQQB
gjcCARUDAgeAMB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDAzA1BgNVHQEE
LjAsgBA0jOnSSuIHYmnVryHAdywMoRUwEzERMA8GA1UEAxMIQXRsYW50aXOCASow
CQYFKw4DAh0FAANBAKi6HRBaNEL5R0n56nvfclQNaXiDT174uf+lojzA4lhVInc0
ILwpnZ1izL4MlI9eCSHhVQBHEp2uQdXJB+d5Byg=
-----END CERTIFICATE-----
```

5.3. 文件扩展名

尽管 PKIX 结构的文本编码可以出现在任何地方, 但已知许多工具在序列化 PKIX 结构时会提供输出此编码的选项. 为了促进互操作性并将 DER 编码与文本编码分开, 证书的文本编码应使用 ".crt" 扩展名. 实现应意识到, 尽管有此建议, 许多工具仍默认使用 ".cer" 扩展名对证书进行此类文本编码.

本节不会以任何方式干扰官方的 application/pkix-cert 注册 [RFC2585] (其中规定 "每个 '.cer' 文件包含一个 DER 格式编码的证书"), 而仅是阐明一种广泛存在的事实上的替代方案.

> 证书链 (Certificate chain) 通常是多个证书的集合, 但似乎并没有一个统一的标准来表示它.
>
> 实践上, 可以将多个 PEM 编码的证书直接串联在一个文件中, [RFC7468] 也允许这么干.

### PKCS #10 标准

> 这个标准是为证书请求语法 (Certificate Request Syntax) 准备的, 不过笔者个人没怎么见过, 但还是顺带说一下了.
>
> 以下内容摘抄自 [RFC7468] Section 7.

PKCS #10 证书请求使用 "CERTIFICATE REQUEST" 标签进行编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 CertificationRequest 结构, 如 [RFC2986] 所述.

```
-----BEGIN CERTIFICATE REQUEST-----
MIIBWDCCAQcCAQAwTjELMAkGA1UEBhMCU0UxJzAlBgNVBAoTHlNpbW9uIEpvc2Vm
c3NvbiBEYXRha29uc3VsdCBBQjEWMBQGA1UEAxMNam9zZWZzc29uLm9yZzBOMBAG
ByqGSM49AgEGBSuBBAAhAzoABLLPSkuXY0l66MbxVJ3Mot5FCFuqQfn6dTs+9/CM
EOlSwVej77tj56kj9R/j9Q+LfysX8FO9I5p3oGIwYAYJKoZIhvcNAQkOMVMwUTAY
BgNVHREEETAPgg1qb3NlZnNzb24ub3JnMAwGA1UdEwEB/wQCMAAwDwYDVR0PAQH/
BAUDAwegADAWBgNVHSUBAf8EDDAKBggrBgEFBQcDATAKBggqhkjOPQQDAgM/ADA8
AhxBvfhxPFfbBbsE1NoFmCUczOFApEuQVUw3ZP69AhwWXk3dgSUsKnuwL5g/ftAY
dEQc8B8jAcnuOrfU
-----END CERTIFICATE REQUEST-----
```

标签 "NEW CERTIFICATE REQUEST" 也被广泛使用. 符合本文档的生成器**必须**生成 “CERTIFICATE REQUEST” 标签. 解析器**可以**将 "NEW CERTIFICATE REQUEST" 视为等同于 "CERTIFICATE REQUEST".

---

## 小结

阅读完本文, 我们应当已经熟悉了对称加密和非对称密码学(公钥密码学)的基本概念, 以及下面这些术语:

1. RSA 公钥密码系统
1. ECC 公钥密码系统

  使用到的椭圆曲线: Curve25519 / Ed25519, Curve448 / Ed448 等

  - X25519 (ECDH) / X448 (EdDH).
  - ECDSA
  - EdDSA (Ed25519 / Ed448).

此外, 我们还应熟悉 PEM 格式下证书、公私钥的文本编码方式, 以及其与 DER 的关系.

[RFC1421]: https://datatracker.ietf.org/doc/html/rfc1421
[RFC2315]: https://datatracker.ietf.org/doc/html/rfc2315
[RFC2585]: https://datatracker.ietf.org/doc/html/rfc2585
[RFC2986]: https://datatracker.ietf.org/doc/html/rfc2986
[RFC5208]: https://datatracker.ietf.org/doc/html/rfc5208
[RFC5280]: https://datatracker.ietf.org/doc/html/rfc5280
[RFC5652]: https://datatracker.ietf.org/doc/html/rfc5652
[RFC5958]: https://datatracker.ietf.org/doc/html/rfc5958
[RFC7468]: https://datatracker.ietf.org/doc/html/rfc7468
