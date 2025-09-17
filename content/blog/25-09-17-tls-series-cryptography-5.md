+++
title = "从零开始了解 TLS 1.3 系列笔记 (六) —— 实用密码学之「对称加密和非对称密码学」"
date = "2025-09-17"
# updated = ""
description = "本文介绍了对称加密和非对称密码学, 重点介绍了 AEAD 加密方案, 以及 ECC 公钥密码系统的应用 (ECDH, EdDSA)."

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

> 一些密码学术语需要我们了解
>
> 1. 加密 (encrypt), 解密 (decrypt)
> 1. 密文 (ciphertext), 明文 (plaintext)
> 1. 对称加密 (symmetric encryption), 非对称加密 (asymmetric encryption)
> 1. cipher: 一般指 "加密算法", 但有些翻译会直译为 "密码", 需要和 "password" 区分.

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

我们前面已经提到过 AE(AD), Authenticated Encryption (with Associated Data), (带关联数据的)认证加密, 其包含了 "认证" 和 "加密" 两大部分. 正确实现了消息认证的对称加密方案, 如 AES-256-CTR-HMAC-SHA256, AES-128-GCM, AES-256-GCM 或 ChaCha20-Poly1305, 均可称为 AE(AD) 加密方案.

1. 认证 (Authentication): 通常使用 HMAC, Poly1305, 或者 GCM 分组模式自带.
2. 加密 (Encryption): 使用对称加密算法, 如 AES 或 ChaCha.

今天的大多数应用程序应该优先选用 AES-128-GCM, AES-256-GCM 或 ChaCha20-Poly1305 加密方案进行对称加密, 而不是自己造轮子.

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

安全的:

- AES (Advanced Encryption Standard, 高级加密标准): 目前最流行的对称加密算法, 由 NIST 于 2001 年发布. 支持 128, 192 和 256 位密钥长度. AES 是一种分组密码算法, 块大小为 128 位, 使用 128 位的 IV. AES-GCM 是目前最常用的对称加密方案之一.
- ChaCha: Salsa 改进版变种, 包括 Chacha20 等. 使用 128 / 256 位密钥和 64 位 IV. ChaCha20-Poly1305 也是一种流行的对称加密方案, 结合了 ChaCha20 流密码和 Poly1305 消息认证码.
- 其他: 包括但不限于 RC5, RC6, 韩国的 ARIA (类似 AES), 中国的 SM4 (类似 AES) 等.

弃用的:

- DES
- 3DES
- RC4
- Blowfish
- ...

## 公钥密码学

1. 公钥密码系统的密钥始终以公钥/私钥对(**密钥对**, Key Pair)的形式出现, 公钥密码系统提供数学框架和算法来生成密钥对. 公钥通常与所有人共享, 而私钥则保密. 公钥密码系统在设计时就确保了在预期的算力下, 几乎不可能从其公开的公钥逆向演算出对应的私钥.
1. 公钥密码系统主要有三大用途: **加密与解密**、**签名与验证**、**密钥交换**. 每种算法都需要使用到公钥和私钥, 比如由公钥加密的消息只能由私钥解密, 由私钥签名的消息需要用公钥验证. 由于加密解密、签名验证均需要两个不同的密钥, 故公钥密码学也被称为非对称密码学.

比较著名的公钥密码系统有: RSA、ECC、(EC)DH. 不同的公钥密码系统可以提供以下一项或多项功能:

- 执行密钥对生成: 随机生成私钥和对应的公钥;
- 执行加解密: 通过公钥加密, 通过私钥解密;
- 作为密钥交换算法;
- 执行数字签名 (消息认证)：通过私钥对消息进行签名, 通过公钥验证签名.

(EC)DH 公钥密码系统的密钥交换功能我们已经在密钥交换一章讲述过, 后面不再赘述; 公钥密码系统的数字签名功能, 我们会在后文单独介绍.

### 非对称加密

应用公钥密码系统的加解密功能即称非对称加密. 非对称加密安全性相对于对称加密高, 但缺点明显:

- 算法更为复杂, 加解密速度比对称加密慢非常多.
- 只能加密/解密很短的消息.

  如在 RSA 系统中, 输入消息需要被转换为大整数, 例如使用 OAEP 填充, 然后才能被加密为密文. (密文实质上就是另一个大整数.)

  一些非对称密码系统, 如 ECC, 不直接提供加密能力, 需要结合使用更复杂的方案才能实现加解密.

现代密码学实践中, 一般混合使用对称加密和非对称加密, 以发挥各自的优势, 如使用非对称加密方案来安全地生成、交换对称加密所需的共享密钥, 然后使用对称加密方案来加密实际的消息内容.

1. KEM: 仅使用非对称加密算法加密另一个密钥, 实际数据的加解密由该密钥完成. 前面介绍 TLS 1.3 后量子密码学改进时提到过 ML-KEM, 这就是一个典型的 KEM.

   RSA-OAEP, RSA-KEM, ECIES-KEM 和 PSEC-KEM. 都是 KEM 加密方案.

2. IKS: 略.

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

以传统的 RSA PKCS#8 PEM ASN.1 格式编码:

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

以传统的 RSA PKCS#8 PEM ASN.1 格式编码, 看起来更长一些:

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

编程生成 RSA 密钥对此处按下不表, 仅简单介绍利用 OpenSSL 命令行工具生成 RSA 密钥对的方式:

```bash
# 生成 1024 位 RSA 私钥, 仅作示例, 实际应用中请使用至少 3072 位
openssl genrsa -out rsa-private-key.pem 1024
# 从私钥导出公钥
openssl rsa -in rsa-private-key.pem -pubout -out rsa-public-key.pem
```

RSA 数学细节和相对应参数此处不展开讲述, 有兴趣的读者可以参考其他资料.

#### RSA 加解密

RSA 一次只能加密/解密一个(大)整数, 一般采用 OAEP 对消息编码为一个个整数再逐个加解密.

#### RSA 签名与验证

参见后文介绍数字签名算法.

### ECC 公钥密码系统

ECC 椭圆曲线密码学于 1985 年被首次提出, 并于 2004 年开始被广泛应用. ECC 被认为是 RSA 的继任者, 新一代的非对称加密算法. 其最大的特点在于相同密码强度下, ECC 的密钥和签名的大小都要显著低于 RSA. 256bits 的 ECC 密钥, 安全性与 3072bits 的 RSA 密钥安全性相当. ECC 的密钥对生成、密钥交换与签名算法的速度都要比 RSA 快.

ECC 中的椭圆曲线可以以多种形式呈现, 这些形式被证明是**同构**的:

- 魏尔施特拉斯 (Weierstrass) 形式: 这是最常见的椭圆曲线形式, 其方程为 $y^2=x^3+ax+b$

  如: $y^2=x^3+7$ (secp256k1, 加密货币领域常用).

  > 需要指出, NSA 推荐的 secp256r1 (NIST P-256, prime256v1, nist256p1) 和 secp256k1 名称只有一字之差, k 和 r 的不同. 其中 k 表示 Koblitz (ECC 发明人), 而 r 表示随机, 即参数是随机选取的, 但美国国家安全局并没有公布随机数的挑选规则, 一直有质疑称 NSA 可能对随机数动过手脚, 让破解难度大幅降低.

- 蒙哥马利 (Montgomery) 形式: 其方程为 $By^2=x^3+Ax^2+x$

  如 Curve25519: $y^2 = x^3+486662x^2+x$.

- 爱德华兹 (Edwards) 形式: 其方程为 $x^2+y^2=1+dx^2y^2$

  如 Ed448: $x^2+y^2=1-39081x^2y^2$. (也可以写成蒙哥马利形式, 称 Curve448).
  
  一般而言, Edwards 形式性能更好, 如 Ed25519 是基于 Curve25519 同构地 "扭曲" 而来, 专为计算速度优化而设计的.

我们推荐使用 Curve52219 / Ed25519 以及 Curve448 / Ed448, 其性能和安全性均优于 NSA 推荐的 NIST P-256 (secp256r1), NIST P-384 (secp384r1), NIST P-521 (secp521r1) 等曲线.
参见 <https://safecurves.cr.yp.to>.

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

即使用 RSA 公钥密码系统的数字签名算法. 刚刚好, 哈希值就是一个大整数.

### ECC 数字签名算法

即使用 ECC 公钥密码系统的数字签名算法. ECC 相对于 RSA, 在相同的安全强度下, 使用更短的密钥长度, 因此在计算和存储方面更为高效. 我们常听说 RSA 证书或 ECC 证书, 就是指内置 RSA 或 ECC 公钥的证书.

常用的 ECC 数字签名算法包括:

1. ECDSA: 在经典 Weierstrass 形式的有限域上使用加密椭圆曲线.

   出于特定实体影响, 目前的 ECC 证书一般使用 ECDSA 数字签名算法.

   需要指出, DSA 是 digital signature algorithm (数字签名算法) 的缩写, 但也可能特指 FIPS 186 Digital Signature Standard (DSS) 中定义的数字签名算法. 一般地, ECDSA 特指 NIST DSA 的使用了 NSA 推荐的 NIST P-256, P-384 或 P-521 曲线的椭圆曲线实现.
1. EdDSA: 在 Edwards 形式的有限域上使用加密椭圆曲线.

   常用变体包括 Ed25519 或 Ed448.

我们推荐使用 EdDSA 作为数字签名算法, 原因前面已经提到过.

## 补充内容: PKIX、PKCS 和 CMS 结构的文本编码 (RFC 7468)

本文档描述并讨论公钥基础设施 X.509 (Public Key Infrastructure X.509, PKIX)、公钥加密标准 (Public-Key Cryptography Standards, PKCS) 和加密消息语法 (Cryptographic Message Syntax, CMS) 的文本编码. 文本编码是众所周知的, 由多个应用程序和库实现, 并被广泛部署. 本文档阐明了现有实现的实际操作规则, 并对其进行了定义, 以便将来的实现能够进行互操作.

### 1. 引言

互联网上使用的若干安全相关标准定义了 ASN.1 数据格式, 这些格式通常采用 BER (Basic Encoding Rules, 基本编码规则) 或 DER (Distinguished Encoding Rules, 可辨别编码规则) [X.690] 格式进行编码, 它们均属于二进制编码. 本文档涉及以下格式的文本编码:

1. 互联网 X.509 公钥基础设施证书及证书吊销列表 (Certificate Revocation Lists, CRL) 规范 [RFC5280] 中的证书、CRLs 以及主体公钥信息结构.
1. PKCS #10: 证书请求语法 [RFC2986].
1. PKCS #7: 加密消息语法 [RFC2315].
1. CMS (加密消息语法) [RFC5652].
1. PKCS #8: 私钥信息语法 [RFC5208]. 在非对称密钥包 [RFC5958] 中更名为 "单非对称密钥", 以及同一文档中的加密私钥信息语法.
1. 授权用互联网属性证书规范 [RFC5755] 中的属性证书.

由于缺乏协调或疏忽等原因, 许多 PKIX、PKCS 和 CMS 库实现了一种与 PEM 编码相似但并非完全相同的基于文本的编码. 本文档规定了*文本编码*格式, 阐述了大多数实现所遵循的事实规则, 并提供了促进未来互操作性的建议. 本文档还为语法元素提供了通用术语, 反映了这一事实标准格式的演变. Peter Gutmann 的*X.509 风格指南*中包含一节 "Base64 编码", 描述了相关格式并提供了与本文件内容相似的建议. 所有图示均为真实、功能性示例, 密钥长度和内部内容尽可能选择最小化.

本文档中的关键词 "必须" ("MUST" / "SHALL" / "REQUIRED")、"绝不能" ("MUST NOT" / "SHALL NOT"), "推荐" ("SHOULD" / "RECOMMENDED")、"不推荐" ("SHOULD NOT" / "NOT RECOMMENDED"), "可以/可能/可选" ("MAY" / "OPTIONAL") 按照 BCP 14, RFC 2119 中的描述进行解释.

### 2. 通用注意事项

文本编码以一行包含 "`-----BEGIN `"、标签和 "`-----`" 开始, 并以一行包含 "`-----END `"、标签和 "`-----`" 结束. 在这些行 (或称"封装边界") 之间, 是根据 RFC4648 第 4 节进行 Base64 编码的数据 (PEM [RFC1421] 称这部分数据为 "封装文本部分"). 封装边界之前允许存在数据, 解析器在处理此类数据时**绝不能**出错. 此外, 解析器**必须**忽略空白字符及其他非 Base64 字符, 且**必须**支持不同的换行规范.

编码数据的类型由 "`-----BEGIN `" (前置封装边界) 行中的类型标签决定. 例如, "`-----BEGIN CERTIFICATE-----`" 表示内容是一个 PKIX 证书 (详见下文). 生成器**必须**在 "`-----END `" (后置封装边界) 行使用与对应 "`-----BEGIN `" 行相同的标签. 标签由区分大小写的、全大写的零个或多个字符组成; 不得包含连续空格或连字符, 首尾也不得出现空格或连字符. 若标签不匹配, 解析器**可以**选择忽略后置封装边界中的标签而非报错: 现存实现中部分要求标签匹配, 部分不作要求.

"BEGIN" 或 "END" 与标签之间必须严格用一个空格字符 (SP) 分隔. 封装边界两端的连字符 (即短横线 "-") 必须正好五个, 不能多也不能少.

标签类型暗示编码数据遵循特定语法. 解析器**必须**能够优雅处理不符合规范的数据. 但需注意本文档发布前的解析器或生成器行为并不统一. 合法的解析器**可能**将内容解释为其他标签类型, 但应注意安全考量章节讨论的潜在风险. 本文档描述的标签标识的容器格式不与特定加密算法绑定, 这符合算法敏捷性原则. 这些格式采用 [RFC5280] 第 4.1.1.2 节所述的 ASN.1 AlgorithmIdentifier 结构.

与传统 PEM 编码 [RFC1421]、OpenPGP ASCII armor 及 OpenSSH 密钥文件格式不同, 文本编码*不*定义也不允许在数据旁编码头信息. 前置封装边界与 Base64 数据之间允许存在空白, 但生成器**绝不能**产生此类空白 (保留此空白区域是对 PEM "封装头部分" 定义的沿袭).

(余下略)

### 3. ABNF

(略)

### 4. 指引

为方便起见, 下列图表概括了后续章节中的结构、编码及参考文献:

| Sec. | 标签名 | ASN.1 类型 | 参考文献 | 模块 |
|:-:|:-:|:-:|:-:|:-:|
| 5 | CERTIFICATE | Certificate | [RFC5280] | id-pkix1-e |
| 6 | X509 CRL | CertificateList | [RFC5280] | id-pkix1-e |
| 7 | CERTIFICATE REQUEST | CertificationRequest | [RFC2986] | id-pkcs10 |
| 8 | PKCS7 | ContentInfo | [RFC2315] | id-pkcs7* |
| 9 | CMS | ContentInfo | [RFC5652] | id-cms2004 |
| 10 | PRIVATE KEY | PrivateKeyInfo (OneAsymmetricKey) | [RFC5208] | id-pkcs8 |
| 11 | ENCRYPTED PRIVATE KEY | EncryptedPrivateKeyInfo | [RFC5958] | id-aKPV1 |
| 12 | ATTRIBUTE CERTIFICATE | AttributeCertificate | [RFC5755] | id-acv2 |
| 13 | PUBLIC KEY | SubjectPublicKeyInfo | [RFC5280] | id-pkix1-e |

(余下略)

### 5. 文本编码: 证书 (Certificate)

#### 5.1. 编码

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

#### 5.2. 说明性文本

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

#### 5.3. 文件扩展名

尽管 PKIX 结构的文本编码可以出现在任何地方, 但已知许多工具在序列化 PKIX 结构时会提供输出此编码的选项. 为了促进互操作性并将 DER 编码与文本编码分开, 证书的文本编码应使用 ".crt" 扩展名. 实现应意识到, 尽管有此建议, 许多工具仍默认使用 ".cer" 扩展名对证书进行此类文本编码.

本节不会以任何方式干扰官方的 application/pkix-cert 注册 [RFC2585] (其中规定 "每个 '.cer' 文件包含一个 DER 格式编码的证书"), 而仅是阐明一种广泛存在的事实上的替代方案.

### 6. 文本编码: CRL

(略)

### 7. 文本编码: PKCS #10 证书请求语法 (Certificate Request Syntax)

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

### 8. 文本编码: PKCS #7 加密消息语法 (Cryptographic Message Syntax)

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

标签 "CERTIFICATE CHAIN" 曾用于表示仅包含证书列表的退化 PKCS #7 结构 (参见 [RFC2315] 第 9 节). 多个现代工具已不再支持此标签. 生成器**绝不能**生成 "CERTIFICATE CHAIN" 标签. 解析器**绝不能**将 "CERTIFICATE CHAIN" 视为等同于 "PKCS7". PKCS #7 是一个已被 CMS [RFC5652] 长期取代的旧规范. 在 CMS 可作为替代方案时, 实现**绝不能**生成PKCS #7.

### 9. 文本编码: CMS (Cryptographic Message Syntax)

CMS 结构使用 "CMS" 标签进行编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 ContentInfo 结构, 如 [RFC5652] 所述.

```
-----BEGIN CMS-----
MIGDBgsqhkiG9w0BCRABCaB0MHICAQAwDQYLKoZIhvcNAQkQAwgwXgYJKoZIhvcN
AQcBoFEET3icc87PK0nNK9ENqSxItVIoSa0o0S/ISczMs1ZIzkgsKk4tsQ0N1nUM
dvb05OXi5XLPLEtViMwvLVLwSE0sKlFIVHAqSk3MBkkBAJv0Fx0=
-----END CMS-----
```

CMS 是 IETF 对 PKCS #7 的继承. [RFC5652] 第1.1.1节描述了自 PKCS #7 v1.5 以来的变更. 实现方案在可选时**必须**生成 CMS 格式, 以促进互操作性和前向兼容性.

### 10. 文本编码: 非对称密钥与 PKCS #8 私钥信息

未加密的 PKCS #8 私钥信息语法结构 (PrivateKeyInfo, 现更名为非对称密钥包 OneAsymmetricKey) 使用 "PRIVATE KEY" 标签编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 PrivateKeyInfo 结构, 如 PKCS #8 [RFC5208] 所述, 或 OneAsymmetricKey 结构, 如 [RFC5958] 所述. 两者语义相同, 可通过版本号区分.

```
-----BEGIN PRIVATE KEY-----
MIGEAgEAMBAGByqGSM49AgEGBSuBBAAKBG0wawIBAQQgVcB/UNPxalR9zDYAjQIf
jojUDiQuGnSJrFEEzZPT/92hRANCAASc7UJtgnF/abqWM60T3XNJEzBv5ez9TdwK
H0M6xpM2q+53wmsN/eYLdgtjgBd3DBmHtPilCkiFICXyaA8z9LkJ
-----END PRIVATE KEY-----
```

### 11. 文本编码: 加密的 PKCS #8 私钥信息

加密的 PKCS #8 私钥信息语法结构 (EncryptedPrivateKeyInfo, [RFC5958]中名称相同) 使用 "ENCRYPTED PRIVATE KEY" 标签编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 PrivateKeyInfo 结构, 如 PKCS #8 [RFC5208] 和[RFC5958] 所述.

```
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIHNMEAGCSqGSIb3DQEFDTAzMBsGCSqGSIb3DQEFDDAOBAghhICA6T/51QICCAAw
FAYIKoZIhvcNAwcECBCxDgvI59i9BIGIY3CAqlMNBgaSI5QiiWVNJ3IpfLnEiEsW
Z0JIoHyRmKK/+cr9QPLnzxImm0TR9s4JrG3CilzTWvb0jIvbG3hu0zyFPraoMkap
8eRzWsIvC5SVel+CSjoS2mVS87cyjlD+txrmrXOVYDE+eTgMLbrLmsWh3QkCTRtF
QC7k0NNzUHTV9yGDwfqMbw==
-----END ENCRYPTED PRIVATE KEY-----
```

### 12. 文本编码: 属性证书 (Attribute Certificate)

(略)

### 13. 文本编码: 公钥 (Public Key)

公钥信息结构 (SubjectPublicKeyInfo) 使用 "PUBLIC KEY" 标签进行编码. 编码内容**必须**是 BER (强烈推荐 DER 格式) 编码的 ASN.1 SubjectPublicKeyInfo 结构, 如 [RFC5280] 第 4.1.2.7 节所述.

```
-----BEGIN PUBLIC KEY-----
MHYwEAYHKoZIzj0CAQYFK4EEACIDYgAEn1LlwLN/KBYQRVH6HfIMTzfEqJOVztLe
kLchp2hi78cCaMY81FBlYs8J9l7krc+M4aBeCGYFjba+hiXttJWPL7ydlE+5UG4U
Nkn3Eos8EiZByi9DVsyfy9eejh+8AXgp
-----END PUBLIC KEY-----
```

(余下略)

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

[X.509SG]: https://www.rfc-editor.org/rfc/rfc7468.html#section-2
[X.690]: https://www.itu.int/rec/T-REC-X.690
[RFC1421]: https://datatracker.ietf.org/doc/html/rfc1421
[RFC2315]: https://datatracker.ietf.org/doc/html/rfc2315
[RFC2585]: https://datatracker.ietf.org/doc/html/rfc2585
[RFC2986]: https://datatracker.ietf.org/doc/html/rfc2986
[RFC5208]: https://datatracker.ietf.org/doc/html/rfc5208
[RFC5280]: https://datatracker.ietf.org/doc/html/rfc5280
[RFC5652]: https://datatracker.ietf.org/doc/html/rfc5652
[RFC5755]: https://datatracker.ietf.org/doc/html/rfc5755
[RFC5958]: https://datatracker.ietf.org/doc/html/rfc5958
