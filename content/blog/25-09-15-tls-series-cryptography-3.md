+++
title = "从零开始了解 TLS 1.3 系列笔记 (四) —— 实用密码学之「HMAC、HKDF 与安全随机数」"
date = "2025-09-15"
updated = "2025-09-15"
description = "本文大部分参考自 Practical Cryptography for Developers Book"

[taxonomies]
tags = ["TLS", "Cryptography"]

[extra]
toc = true
katex = true
tldr = """
1. MAC 用以校验消息完整性, HMAC 是一种常用的 MAC 算法.
1. AEAD 利用 HMAC 来校验解密后的消息完整性, 以及认证通信双方持有相同的共享密钥. 如果通信双方持有不同的共享密钥 (即未认证, 而 AE 代表 Authenticated Encryption, 经过认证的加密), 或中间人篡改了密文或关联数据 (即 AD, Associated Data), 则计算得到的 MAC 码不匹配.
1. HMAC 还可应用于密钥派生, 即 HKDF. 依据 RFC 5869, HKDF 遵循 "提取然后扩展" (extract-then-expand) 的范式. 第一阶段, 从不定长的低强度的 IKM (Input Key Material, 输入密钥材料) 出发, 引入盐值作密钥, 使用 HMAC 计算 "提取" (extract) 出一个固定长度的高强度的 PRK (Pseudo-Random Key, 伪随机密钥). 第二阶段, 将 PRK 扩展 (expand) 为所需长度的 OKM (Output Key Material, 输出密钥材料).
"""
+++

## 导语

MAC (Message Authentication Code, 消息认证码)、HMAC (Hash-based MAC, 哈希消息认证码)和 KDF (**Key Derivation** Function, **密钥派生**函数) 在密码学中发挥着重要作用. 让我们解释一下何时需要 MAC, 如何计算 HMAC 以及它与 KDF 的关系.

### MAC

MAC 由给定密钥和消息计算而来:

```
MAC = f(key, message)
```

通常, 它的表现类似于加密哈希函数:

- **难以分析**: 密钥或消息的微小变化会生成完全不同的 MAC 值.
- **不可逆**: 从其哈希值逆向演算出输入值应该是不可行的. 这意味着没有比暴力破解更好的破解方法.
- **抗碰撞**: 几乎不可能找到具有相同哈希值的两条不同消息.

例如, MAC 码可以通过 HMAC-SHA256 算法计算, 如下所示:

```
HMAC-SHA256('key', 'some msg') = 32885b49c8a1009e6d66662f8462e7dd5df769a7b725d1d546574e6d5d6e76ad
```

MAC 码是*数字真实性代码* (digital authenticity code), 类似于数字签名 (digital signature), 但具有预共享密钥 (pre-shared key).

### MAC 算法

包括但不限于:

- [HMAC](https://en.wikipedia.org/wiki/HMAC) (Hash-based MAC, 哈希消息认证码), 如 HMAC-SHA256;
- [KMAC](https://www.cryptosys.net/manapi/api_kmac.html) (Keccak-based MAC, 基于 Keccak 的消息认证码);
- 基于对称密码算法的 MAC, 如 [**Poly1305**](https://en.wikipedia.org/wiki/Poly1305) (Bernstein 一次性身份验证) 等等.
- 其他, 如 [SipHash](https://en.wikipedia.org/wiki/SipHash) (一种快速的、加密的哈希函数, 适用于哈希表).

### MAC 主要应用

MAC 码主要用于验证消息未被篡改. 如当通信双方持有相同的共享密钥时, 发送方可以计算消息的 MAC 码并将其附加到消息中; 接收方收到消息后, 使用相同的密钥计算消息的 MAC 码, 并将其与附加的 MAC 码进行比较; 如果两个 MAC 码匹配, 则消息被认为是完整且未被篡改的.

### 认证加密 (Authenticated Encryption, AE)

使用 MAC 码的另一种场景是认证加密.

加密方:

1. 首先, 我们派生一密钥.
1. 使用该密钥加密消息.
1. 使用该密钥和**原始消息**计算 MAC 码, 并将其附加到第二步的输出中.

接收方:

1. 使用一密钥解密密文, 此密钥并不一定是正确的.
2. 使用该密钥和解密结果计算 MAC 码, 并将其与附加的 MAC 码进行比较; 如果两个 MAC 码匹配, 则密钥一致, 解密成功且密文未被篡改; 否则, 密钥不正确或密文被篡改.

> 背景知识
>
> 可能大家并不熟悉 AE 这个术语, 但 AEAD (Authenticated Encryption with Associated Data, 带关联数据的认证加密) 应该就是熟面孔了. AEAD 是 AE 的一种扩展, 允许在认证加密过程中包含未加密的关联数据 (Associated Data, AD), 如路由信息等需要中间盒处理的信息. 例如, TLS 1.3 中使用的 AES-{128|256}-GCM 和 ChaCha20-Poly1305 都是 AEAD 加密方案.
>
> AEAD 的具体做法我们不需要了解, 此处按下不表, 我们了解如何使用即可.

由此, 我们同时验证了通信双方持有相同的共享密钥 (即认证, AE 代表 Authenticated Encryption, 经过认证的加密), 以及消息未被篡改 (完整性).

### 基于 MAC 的伪随机生成函数 (PRF, Pseudo-Random Function)

MAC 码的另一个应用是作 PRF. 我们可以从某个盐和一个随机数(种子)开始.我们可以计算 `next_seed` 如下:

```
next_seed = MAC(salt, seed)
```

每次计算上述公式后, 下一个伪随机数都会被 "随机变化", 我们可以用它来生成一定范围内的下一个随机数.

## HMAC 和密钥派生

简单计算 `MAC = H(key || msg)` 来获取 MAC [被认为是不安全的](https://en.wikipedia.org/wiki/HMAC#Design_principles). 建议改用 HMAC 算法, 例如 HMAC-SHA256.

> 背景知识
>
> 1. 大多数哈希函数存在长度拓展攻击, 即在不知道密钥的情况下, 可以通过已知的 `H(key || msg)` 来计算有效的 `H(key || msg || extra)`;
> 1. 计算 `MAC = H(msg || key)`, 则存在哈希冲突的可能性;
> 1. 计算 `MAC = H(key || msg || key)` 仍然被证明是不安全的;
> 1. HMAC 计算 `HMAC = H(key || H(key || msg))`, 采用双重哈希, 消除 msg 长度影响, 目前尚未被证明不安全; 对于 SHA-3 (Keccak) 则不需要双重哈希, 只需 `HMAC = H(key || msg)`, 因为 SHA-3 不易受长度拓展攻击.

### [什么是 HMAC](https://en.wikipedia.org/wiki/HMAC#Definition)

HMAC (Hash-based Message Authentication Code, 哈希消息认证码) 是一种基于哈希函数的 MAC 算法. 它使用一个密钥和一个消息, 通过两次哈希计算得到 MAC 码.

$$\operatorname {HMAC} (K,m)=\operatorname {H} {\Bigl (}{\bigl (}K'\oplus opad{\bigr )}\parallel \operatorname {H} {\bigl (}\left(K'\oplus ipad\right)\parallel m{\bigr )}{\Bigr )}$$

其中:

- $\operatorname {H}$ 是 HMAC 过程所使用的加密哈希函数, 如 SHA-256;
- $m$ 是消息;
- $K$ 是一个密钥;
- $K'$ 是密钥 $K$ 经过右侧补 0 填充或哈希后(必要时右侧补 0)的结果, 以确保其长度等于哈希函数的块大小 (block size);
- $ipad$ 和 $opad$ 分别是内填充 (inner padding) 和外填充 (outer padding), 长度与块大小相同, 分别由值为 0x5c, 0x36 的重复字节组成;
- $\oplus$ 表示按位异或 (XOR) 操作;
- $\parallel$ 表示拼接.

HMAC 用于验证消息真实性和完整性, 有时还用于密钥派生.

### KDF

KDF (Key Derivation Function, 密钥派生函数) 是一种将可变长度密码转换为固定长度密钥的函数:

```
key = KDF(password)
```

我们可以使用 SHA-256 对密码进行哈希处理, 使用哈希值作密钥, 这就是一个简单的 KDF. 但请不要这样做, 因为它不安全: 简单哈希容易受到字典攻击 (查表法).

为此, 我们可以引入盐 (salt) 执行 HMAC 操作, 此即 **HKDF** (HMAC-based KDF), 具体内容见下文.

使用 HKDF 进行密钥派生不如现代 KDF 安全, 如 PBKDF2, Bcrypt, Scrypt, Argon2 等.

## KDF: 从密码(或密钥)派生密钥

PBKDF2, Bcrypt, Scrypt, Argon2 等 KDF 函数, 通过引入盐并结合多次迭代来增加密码破解的难度, 迭代过程也称 [key stretching (密钥拉伸)](https://en.wikipedia.org/wiki/Key_stretching).

要计算安全的 KDF, 需要一些 CPU 时间来派生密钥 + 一些内存. 因此, 派生密钥是计算成本高昂的, 因此密码破解的尝试也会面临高昂计算成本.

当现代 KDF 与适当的配置参数一起使用时, 破解密码的速度会很慢 (例如, 每秒只能作 5-10 次尝试, 而不是每秒数千或数百万次尝试).

上述现代 KDF 均未申请专利, 可公开使用.

由于本系列主要介绍 TLS 1.3, 因此不再赘述现代 KDF 的细节 (我们只用到了 HKDF), 有兴趣的读者可参考 [Practical Cryptography for Developers Book](https://practicalcryptography.dev/). 下一节对 HKDF 的介绍, 主要翻译自 [RFC 5869](https://datatracker.ietf.org/doc/html/rfc5869).

## HKDF (RFC 5869)

本文档规定了一个简单的基于 HMAC 的 KDF, 它可以作为各种协议和应用程序中的 (加密) 组件. KDF 旨在支持广泛的应用程序和需求, 并在使用加密哈希函数方面是保守的.

### 1. 引言

KDF 是加密系统的基本和必要组件. 其目标是获取一些初始密钥材料源, 并从中派生出一个或多个具备强加密强度的密钥.

本文档规定了一个简单的基于 HMAC 的 KDF, 名为 HKDF, 它可以作为各种协议和应用程序中的 (加密) 组件, 并且已经在几个 IETF 协议中使用, 包括 IKEv2、PANA 和 EAP-AKA. 本文档旨在以通用化的方式, 促进其在未来协议和应用中的采纳, 并防止多种 KDF 机制的无序发展. 它不是要求更改现有协议的呼吁, 也不会更改或更新使用此 KDF 的现有规范.

HKDF 遵循 "提取然后扩展" (extract-then-expand) 的范式, 其中 KDF 在逻辑上由两个模块组成. 第一阶段获取输入密钥材料并从中 "提取" 固定长度的伪随机密钥 K. 第二阶段将密钥 K "扩展" 为几个额外的伪随机密钥 (KDF 的输出).

在许多应用程序中, 输入密钥材料不一定均匀分布, 攻击者可能对其有部分了解 (例如, 由密钥交换协议计算的 Diffie-Hellman 值), 甚至对其有部分控制 (如在一些熵收集应用程序中). 因此, "提取" 阶段的目标是将输入密钥材料可能分散的熵 "集中" 到一个短而高强度的**伪随机密钥** (pseudorandom key, **PRK**) 中. 在一些应用程序中, 输入可能已经是一个良好的 PRK; 在这些情况下, "提取" 阶段不是必需的, "扩展" 部分可以单独使用. 第二阶段, 将伪随机密钥 "扩展" 到所需长度; 输出密钥的数量和长度取决于需要密钥的特定加密算法.

请注意, 一些现有的 KDF 规范要么只考虑第二阶段, 要么没有明确区分 "提取" 和 "扩展" 阶段, 经常导致设计缺陷. 本规范的目标是适应广泛的 KDF 需求, 同时最小化对底层哈希函数的假设. 提取然后扩展" 范式很好地支持了这一目标.

### 2. HKDF

#### 2.1. 符号系统

HMAC-Hash 表示使用哈希函数 "Hash" 的 HMAC 函数. HMAC 始终有两个参数: 第一个是密钥, 第二个是输入(或消息). (请注意, 在提取步骤中,  IKM 用作 HMAC 输入, 而不是 HMAC 密钥.)

当消息由多个元素组成时, 我们在第二个参数中将其串接 (表示为 |); 例如, HMAC(K, elem1 | elem2 | elem3).

本文档中的关键词 "必须" ("MUST" / "SHALL" / "REQUIRED")、"绝不能" ("MUST NOT" / "SHALL NOT"), "推荐" ("SHOULD" / "RECOMMENDED")、"不推荐" ("SHOULD NOT" / "NOT RECOMMENDED"), "可以/可能/可选" ("MAY" / "OPTIONAL") 按照 BCP 14, RFC 2119 中的描述进行解释

#### 2.2. 第一步: 提取 (extract)

```
HKDF-Extract(salt, IKM) -> PRK
```

可选:

- `Hash`: 用于 HMAC 的哈希函数, HashLen 是其输出长度 (以字节为单位);

入参:

- `salt`: 可选的随机盐值 (一个非秘密随机值); 如果未提供, 则将其设置为 HashLen 字节的零值;
- `IKM`: 输入密钥材料 (input key material);

输出:

- `PRK`: 输出的长度为 HashLen 字节的伪随机密钥 (pseudorandom key).

PRK 由以下公式计算:

```
PRK = HMAC-Hash(salt, IKM)
```

#### 2.3. 第二步: 扩展 (expand)

```
HKDF-Expand(PRK, info, L) -> OKM
```

可选:

- `Hash`: 用于 HMAC 的哈希函数, HashLen 是其输出长度 (以字节为单位);

入参:

- `PRK`: 伪随机密钥 (pseudorandom key), 长度至少为 HashLen 字节 (通常, 这是由 HKDF-Extract 生成的输出);
- `info`: 可选的上下文和应用程序特定信息 (可以是零长度);
- `L`: 所需输出密钥材料 (output keying material) 的长度 (以字节为单位); L 必须小于或等于 255 * HashLen;

输出:

- `OKM`: 输出的密钥材料 (output keying material), 长度为 L 字节;

OKM 由以下公式计算:

```
N = ceil(L/HashLen)
T = T(1) | T(2) | T(3) | ... | T(N)
OKM = first L bytes of T
```

其中:

```
T(0) = empty string (zero length)
T(1) = HMAC-Hash(PRK, T(0) | info | 0x01)
T(2) = HMAC-Hash(PRK, T(1) | info | 0x02)
T(3) = HMAC-Hash(PRK, T(2) | info | 0x03)
...
```

### 3. HKDF 的使用注意事项

本节讨论 HKDF 的一些使用注意事项.

#### 3.1. 关于盐 (salt)

HKDF 被定义为可以在有或没有随机盐值的情况下运行. 这样做是为了适应那些无法获得盐值的应用程序. 然而, 我们强调, 使用盐值可以显著增强 HKDF 的强度, 确保哈希函数不同用途之间的独立性, 支持 "源独立地" 提取, 并强化支持 HKDF 设计的分析结果.

随机盐值与初始密钥材料在两个方面有根本不同: 它不是保密的, 并且可以重复使用. 因此, 许多应用程序都可以应用盐值. 例如, 一个通过将 HKDF 应用于可再生熵池 (例如, 采样的系统事件) 来持续生成输出的伪随机数生成器 (PRNG) 可以固定一个盐值, 并将其用于 HKDF 的多个应用, 而无需保护盐值的秘密性. 在另一个应用领域, 一个从 Diffie-Hellman 交换中导出加密密钥的密钥协商协议可以从通信双方作为密钥协商一部分交换和认证的公开随机数中导出盐值 (这是 IKEv2 中采用的方法).

理想情况下, 盐值是长度为 HashLen 的随机或伪随机字符串. 然而, 即使是质量较低的盐值 (尺寸较短或熵有限) 也可能对输出密钥材料的安全性做出显著贡献; 因此, 鼓励应用程序设计者在应用程序能够获取此类值时, 向 HKDF 提供盐值.

值得注意的是, 虽然不常见, 但某些应用程序甚至可能拥有保密的盐值; 在这种情况下, HKDF 提供了更强的安全保证. 这种应用程序的一个例子是 IKEv1 的 "公钥加密模式", 其中提取器的 "盐值" 是从秘密随机数计算出来的; 类似地, IKEv1 的预共享模式使用从预共享密钥派生的秘密盐值.

#### 3.2. 关于 info

尽管 "info" 值在 HKDF 的定义中是可选的, 但它在应用程序中通常非常重要. 其主要目标是将派生密钥材料与应用程序和上下文特定的信息绑定. 例如, "info" 可能包含协议号、算法标识符、用户身份等. 特别是, 它可以防止为不同上下文派生相同的密钥材料 (当在这些不同上下文中使用相同的 IKM 时) . 如果需要, 它还可以容纳密钥扩展部分的额外输入 (例如, 应用程序可能希望将密钥材料与其长度 L 绑定, 从而使 L 成为 info 字段的一部分) . "info" 有一个技术要求: 它应该独立于 IKM.

#### 3.3. 跳过还是不跳过(提取)

在某些应用程序中, IKM 可能已经作为密码学强度高的密钥存在 (例如, TLS RSA 密码套件中的预主密钥将是一个伪随机字符串, 除了前两个八位字节) . 在这种情况下, 可以跳过提取部分, 直接使用 IKM 作为扩展步骤中的密钥. 另一方面, 为了与一般情况兼容, 应用程序仍可能使用提取部分. 特别是, 如果 IKM 是随机的 (或伪随机的) , 但比 HMAC 密钥长, 则提取步骤可以用于输出一个合适的 HMAC 密钥 (在 HMAC 的情况下, 通过提取器进行这种缩短并非严格必要, 因为 HMAC 也被定义为可以处理长密钥) . 然而, 请注意, 如果 IKM 是 Diffie-Hellman 值, 如同 TLS 与 Diffie-Hellman 的情况, 则不应跳过提取部分. 这样做将导致使用 Diffie-Hellman 值 g^{xy} 本身 (它不是一个均匀随机或伪随机字符串) 作为 HMAC 的密钥 PRK. 相反, HKDF 应该将提取步骤应用于 g^{xy} (最好带有一个盐值) , 并使用生成的 PRK 作为 HMAC 在扩展部分中的密钥.

在所需的密钥位数 L 不超过 HashLen 的情况下, 可以直接将 PRK 截断获得最终的 OKM. 然而, 不推荐这样做, 特别是因为它会省略在派生过程中使用的 "info" (也不建议将 "info" 作为输入添加到提取步骤中) .

#### 3.4. 独立性的作用

KDF 的分析假设 IKM 来自某个源, 该源被建模为特定长度比特流的概率分布 (例如, 由熵池产生的流, 从随机选择的 Diffie-Hellman 指数派生的值等) ; IKM 的每个实例都是该分布中的一个样本. 密钥派生函数的一个主要目标是确保, 当将 KDF 应用于从 (相同) 源分布中抽取的任意两个值 IKM 和 IKM' 时, 生成的密钥 OKM 和 OKM' 在本质上是相互独立的 (在统计或计算意义上) . 为了实现这一目标, KDF 的输入必须从适当的输入分布中选择, 并且输入也必须相互独立选择 (从技术上讲, 即使以 KDF 的其他输入为条件, 每个样本也必须具有足够的熵) .

独立性也是提供给 KDF 的盐值的一个重要方面. 虽然没有必要对盐值保密, 并且同一个盐值可以与多个 IKM 值一起使用, 但假定盐值独立于输入密钥材料. 特别是, 应用程序需要确保盐值不是由攻击者选择或操纵的. 例如, 考虑 (如 IKE 中) 盐值是从密钥交换协议中各方提供的随机数派生的情况. 在协议使用此类盐值派生密钥之前, 它需要确保这些随机数被认证为来自合法方, 而不是由攻击者选择的 (在 IKE 中, 例如, 这种认证是经过认证的 Diffie-Hellman 交换的一个组成部分).

### 4. HKDF 的应用

HKDF 旨在用于各种 KDF 应用. 这些应用包括从弱随机性来源获得伪随机性; 在密钥协商协议中从共享的 Diffie-Hellman 值导出加密密钥; 从混合公钥加密方案导出对称密钥; 用于密钥封装机制的密钥导出; 以及更多. 所有这些应用都可以受益于 HKDF 的简洁性和多功能性, 以及其分析基础.

另一方面, 预计某些应用由于特定的操作要求将无法 "按原样" 使用 HKDF, 或者可以使用但无法充分发挥该方案的全部优势. 一个重要的例子是从低熵源 (例如用户密码) 导出加密密钥. HKDF 中的提取步骤可以集中现有熵, 但不能放大熵. 对于基于密码的 KDF, 主要目标是使用两个要素来减缓字典攻击: 一个盐值, 以及有意减慢密钥导出计算. HKDF 自然支持使用盐; 然而, 减速机制并非本规范的一部分. 对基于密码的 KDF 感兴趣的应用应考虑例如 PKCS5 是否比 HKDF 更能满足其需求.

### 5. 安全注意事项

尽管 HKDF 组成简单, 但在其设计和分析中已考虑了许多安全因素. 对所有这些方面的阐述超出了本文档的范围.

相关论文已投入大量精力, 对 HKDF 作为一种多用途 KDF 进行了密码学分析, 该分析在利用加密哈希函数的方式上非常谨慎. 鉴于我们对当前哈希函数强度的信心有限, 这一点尤为重要. 然而, 这项分析并不意味着任何方案的绝对安全性, 并且它在很大程度上取决于底层哈希函数的强度和建模选择. 尽管如此, 它有力地表明了 HKDF 设计的正确结构及其相对于其他常见 KDF 方案的优势.

(余下内容略去不译)

## 安全随机数

密码学中, 随机性 (熵) 起着非常重要的作用. 在许多算法中, 我们需要随机的 (不可预测的) 数字. 如果这些数字不是不可预测的, 算法就会受到损害.

在计算机科学中, 随机数通常来自伪随机数生成器 (pseudo-random numbers generators, PRNG), 由一些初始随机性初始化, 这些初始随机性一般来自极难预测的用户输入等, 可从操作系统读取, 如 Linux 的 `/dev/random`;
在密码学中, 使用安全 PRNG, 称为 CSPRNG, 它通常将熵与 PRNG 和其他技术相结合, 使生成的随机性在密码学意义上不可预测.

应当注意, 获取安全随机数的开销通常是比较大的, 而许多语言提供的随机数生成函数并非 CSPRNG (例如 Python 的 `random` 库), 而是基于时间戳等可预测的种子获得的伪随机数, 不能用于密码学场景.

---

## 小结

本文介绍了 MAC、HMAC、KDF 和 HKDF 的基本概念及其应用. 除却这四个术语外, 以下再摘抄若干术语、表达式供读者快速回顾:

- `HMAC = H(key || H(key || msg))`;
- **`AEAD`**
- **`PRK = HKDF-Extract(salt, IKM)`**;
- **`OKM = HKDF-Expand(PRK, info, L)`**;

在阅读完本文后, 我们应该对上述表达式及相关术语不再陌生.
