---
title: "RSA 密钥复用：用 GCD 恢复 φ(n)"
date: 2025-01-01
draft: false
description: "当服务器用固定模数 n 生成多组密钥对时，如何通过 GCD 恢复私钥"
categories: ["CTF Writeup"]
tags: ["crypto", "RSA", "GCD"]
---

## 场景

某个 RSA 服务用固定的模数 `n` 反复生成密钥对：你可以提交一个 `d`，服务器返回对应的 `e`（或反过来）。同时给出了一段用特定公钥加密的密文，要求解密。

你拿不到 `p`、`q`，也拿不到目标公钥对应的私钥。但你可以反复获取其他 `(e, d)` 对。

这就够了。

## RSA 回顾

简单过一下关键关系：

- `n = p * q`
- `φ(n) = (p-1)(q-1)`（[欧拉函数](https://en.wikipedia.org/wiki/Euler%27s_totient_function)）
- 公私钥满足 `e * d ≡ 1 (mod φ(n))`
- 等价于 `e * d - 1 = k * φ(n)`，`k` 是某个正整数

## 攻击原理

每组合法的 `(e_i, d_i)` 都满足上面的关系，也就是说 `e_i * d_i - 1` 一定是 `φ(n)` 的整数倍。

收集多组后，对所有 `e_i * d_i - 1` 求 [GCD](https://en.wikipedia.org/wiki/Greatest_common_divisor)：

```
φ(n) | gcd(e_1*d_1 - 1, e_2*d_2 - 1, ...)
```

多取几组（10 组左右），GCD 基本就收敛到 `φ(n)` 本身了。偶尔可能得到 `k * φ(n)`，试着除以小因子即可。

拿到 `φ(n)` 后，对目标 `e` 求模逆得到 `d`，解密。

## 代码

```python
from math import gcd
from Crypto.Util.number import long_to_bytes, inverse

# 从服务器收集到的 (e, d) 对
pairs = [
    (e1, d1),
    (e2, d2),
    # ...
]

# 题目给定
target_e = 65537
n = 0x...  # 模数
c = 0x...  # 密文

# 计算 φ(n)
phi = pairs[0][0] * pairs[0][1] - 1
for e, d in pairs[1:]:
    phi = gcd(phi, e * d - 1)

# 如果 phi > n，可能是 φ(n) 的倍数，尝试缩小
if phi > n:
    for k in range(2, 100):
        if phi % k == 0 and phi // k < n:
            phi = phi // k
            break

# 解密
d = inverse(target_e, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

## 为什么这个攻击能成立

根本原因是 **模数 `n` 被复用了**。所有密钥对共享同一个 `φ(n)`，每多拿一组 `(e, d)` 就多了一个关于 `φ(n)` 的线性约束。GCD 是求公因子最直接的工具。

安全的做法是：每次生成密钥对都用全新的 `n`，即全新的 `p` 和 `q`。

## 小结

- 固定 `n` + 多组密钥对 = `φ(n)` 泄露
- GCD 是恢复 `φ(n)` 的标准手法
- `φ(n)` 在手，RSA 就完全破解了
