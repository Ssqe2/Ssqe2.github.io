---
title: "双重取模方程的多项式求根解法"
date: 2025-01-01
draft: false
description: "把 X % (Y - m) % Z == A 转化为有限域上的多项式方程"
categories: ["CTF Writeup"]
tags: ["crypto", "数论", "有限域", "多项式"]
---

## 问题

给定超大整数 X, Y, Z, A，要求找到满足条件的 m：

```python
m = int(flag.encode().hex(), 16)
assert m < Z
assert X % (Y - m) % Z == A
```

X 约 2055 位十进制，Y 约 360 位，Z 是 77 位素数，A 是 76 位数。题目还附带了一些看起来像提示的字符串。

第一眼看到这题的反应：取模套取模，怎么逆？

## 走过的弯路

题目给了一些看起来像提示的字符串，第一反应是直接拿来试：

```python
candidates = ["modulo_is_fun", "114514", "yyy", ...]
for c in candidates:
    flag = f"flag{{{c}}}"
    m = int(flag.encode().hex(), 16)
    if m < Z and X % (Y - m) % Z == A:
        print(f"Found: {flag}")
```

全部没中。那遍历短 flag 爆破？m < Z 是 77 位数，搜索空间天文数字，也不现实。

又试了从等式反推：`X % (Y - m) = A + k*Z`，遍历 k，再从 `X = q*(Y-m) + A + k*Z` 搜索 q 和 m。但两个未知数一个方程，小范围搜了一圈还是没命中。

卡了很久。

## 回到数据本身

逆向走不通，那就看看数据有没有什么蹊跷。

随手算了一下：

```python
R0 = X % Y  # = 345223，只有6位
```

X 除以 Y 的余数极小。这不正常——两个毫无关系的大数做取模，余数不可能这么小。说明 X 是精心构造的。

## Base-Y 展开

把 X 写成 Y 进制，看看各个"数位"：

```python
digits = []
temp = X
while temp > 0:
    digits.append(temp % Y)
    temp //= Y
```

得到 6 个数位，呈现出清晰的规律：

- 4 个数位很小（5-6 位数）
- 2 个数位非常接近 Y（差值只有几千）

这不是巧合。

## 核心推导

关键等式：**Y ≡ m (mod Y-m)**

这是因为 Y = (Y-m) + m，所以 Y 除以 (Y-m) 余 m。

有了这个，在 mod (Y-m) 的意义下，Y 的幂次都可以用 m 的幂次替换：

```
Y^i ≡ m^i  (mod Y-m)
```

对于接近 Y 的数位 `a_i = Y - c_i`：

```
a_i ≡ m - c_i  (mod Y-m)
```

把 X 的 base-Y 展开代入，所有 Y 替换为 m，得到：

```
X mod (Y-m) = g(m) = 小数·m^5 - 小数·m^4 + ... + 小数
```

系数全都是几千量级的小数。

因为系数很小，g(m) 的值远小于 Y-m，所以 `X % (Y-m)` 就精确等于 `g(m)`，没有额外的整除。可以用随机 m 值验证这一点。

## 有限域上的多项式求根

现在问题变成了：

```
g(m) ≡ A  (mod Z)
```

Z 是素数，所以这是在有限域 [GF(Z)](https://en.wikipedia.org/wiki/Finite_field) 上解 5 次多项式方程。标准工具是 [Cantor-Zassenhaus 算法](https://en.wikipedia.org/wiki/Cantor%E2%80%93Zassenhaus_algorithm)。

用 SageMath：

```python
R.<m> = GF(Z)[]
f = poly - A  # poly 是推导出的多项式
roots = f.roots()
for root, _ in roots:
    try:
        flag = bytes.fromhex(hex(int(root))[2:]).decode()
        if flag.isprintable():
            print(f"Found: {flag}")
    except:
        pass
```

解出 3 个根，其中一个能解码为合法 ASCII。

## 回头看

那些"提示"字符串根本不是 flag 的内容提示，更可能是出题人的恶趣味（或者用来干扰的）。这题从头到尾是纯数学：

1. 发现 X mod Y 极小 → base-Y 展开有特殊结构
2. 利用 Y ≡ m (mod Y-m) 把取模运算转成多项式
3. 有限域上求根

暴力和猜测都是死路，只有认真分析数据结构才能找到突破口。卡了很久才意识到要去看 X 和 Y 的关系，而不是盯着 flag 的内容瞎猜。
