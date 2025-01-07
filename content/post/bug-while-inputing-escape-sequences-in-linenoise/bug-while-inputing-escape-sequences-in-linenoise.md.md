---
title: "使用Linenoise时转义序列显示异常"
description: "Linenoise 转义序列问题的产生原因及其解决方法。"
slug: bug-while-inputing-escape-sequences-in-linenoise
date: 2024-11-07T17:10:33+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
  - Linenoise
  - Shell
categories:
  - 经验分享
---

## Linenoise 简介

[Linenoise](https://github.com/antirez/linenoise/blob/master/linenoise.c) 是一个轻量级的行编辑库，适用于需要简单命令行编辑功能的应用程序，可以用它快速完成 shell 的开发。它支持多平台，并且与 GNU Readline 库相比，Linenoise 的代码更加简洁，易于集成和使用。

## 转义序列显示异常

在移植 Linenoise 之后，发现此时 Shell 存在一个 Bug，长按某些按键时会输出`[D`、`[A`、`D`、`A`之类的乱码，例如，长按左右方向键时会输出如下字符，并且出现的时机不规律。

![alt text](post/bug-while-inputing-escape-sequences-in-linenoise/imgs/img1.png)

## 分析

按理来说按下左右方向键不应输出任何字符，所以肯定和左右方向键的转义序列有关，一些按键的转义序列如下：

- 上箭头键：ESC [ A 或 \x1b[A
- 下箭头键：ESC [ B 或 \x1b[B
- 左箭头键：ESC [ D 或 \x1b[D
- 右箭头键：ESC [ C 或 \x1b[C
- 删除键（Del）：ESC [ 3 ~ 或 \x1b[3~

查看 Linenoise 源码发现，处理逻辑为逐个尝试输入`ESC`、`[`、`A/B/C/D/数字～`

![alt text](post/bug-while-inputing-escape-sequences-in-linenoise/imgs/img2.png)

问题就出在逐个输入这里，如果输入采用的串口不稳定（我这里是 pl011），快速输入就会导致读取失败（Linenoise 读取时串口缓冲区还没有数据），这里直接 break 就会导致状态机跳出当前状态，使得`ESC`、`[`、`A`三者被拆开，并没有当作一个`上箭头键`来处理。

## 解决方案

解决方案也很简单，就是将 break 改成循环读入，如果读取失败就重新尝试。

```c
while (1) {
    if (read(l->ifd, seq, 1) != -1) {
        break;
    }
}
```
