---
title: "无网络环境下的 xhtml2pdf eval 注入：用颜色编码外带数据"
date: 2025-02-16
draft: false
description: "CVE-2023-33733 eval() 注入在断网容器中的利用——通过 PDF 颜色指令逐字节带出文件内容"
categories: ["CTF Writeup"]
tags: ["web", "CVE", "xhtml2pdf", "RCE"]
---

## 背景

这道题和经典的 html2pdf 是同一个漏洞——xhtml2pdf（基于 ReportLab）在解析 `<font color="...">` 时，会对 `eval(...)` 形式的值直接调用 Python 的 `eval()` 执行任意表达式。这就是 CVE-2023-33733。

不同之处在于，容器启动时执行了 `route delete default`，把默认路由删了。没有网络意味着 reverse shell、HTTP 外带、DNS 隧道全部失效。唯一的输出通道就是返回的 PDF 本身。

## 思路

既然 eval 的返回值会被当作颜色字符串处理，而 xhtml2pdf 支持 `#RRGGBB` 格式的颜色——那么只要把想读的文件内容按每 3 字节拼成一个合法的十六进制颜色值，xhtml2pdf 就会老老实实把它写进 PDF 的内容流里。

具体来说：

1. 用 eval 读取目标文件
2. 取出 3 个字节，格式化为 `#XXYYZZ`
3. xhtml2pdf 把这个颜色渲染到 PDF 中，对应的内容流会出现 `R G B rg` 指令
4. 下载 PDF，解码内容流，从 `rg` 指令中还原出原始字节

## 构造 payload

eval 内部不能直接用引号（会和外层的属性引号冲突），所以用 `chr()` 拼接绕过：

```html
<font color="eval(chr(39)+chr(35)+chr(39)+chr(43)+ ... )">X</font>
```

核心表达式拆开来看就是：

```python
'#' + ''.join(hex(b)[2:].zfill(2) for b in open('/data/FLAG','rb').read()[0:3])
```

每次请求带 3 字节，通过调整切片的 offset 逐段提取。flag 通常二十几个字节，跑七八次就够了。

## 从 PDF 中提取数据

xhtml2pdf 默认对内容流做 ASCII85Decode + FlateDecode 双重编码。解码之后是标准的 PDF 绘图指令，其中颜色设置长这样：

```
0.267 0.412 0.639 rg
```

三个浮点数分别是 R、G、B 通道，范围 0-1。乘以 255 取整就能还原出原始字节：

```python
r_byte = round(0.267 * 255)  # 0x44
g_byte = round(0.412 * 255)  # 0x69
b_byte = round(0.639 * 255)  # 0xa3
```

用 Python 写个简单的解析脚本，对每次请求返回的 PDF 提取颜色值，拼起来就是完整的文件内容。

## 小结

这题本质上是经典 CVE 的一个约束变体：堵住了网络出口，迫使你思考"还有什么通道能把数据带出来"。PDF 本身就是输出，颜色值就是天然的数据载体——每次 3 字节不多，但够用。
