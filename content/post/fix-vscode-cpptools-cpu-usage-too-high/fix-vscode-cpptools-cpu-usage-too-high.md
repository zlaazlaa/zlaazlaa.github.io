---
title: "VS Code C/C++ 插件 cpptools CPU 占用过高"
description:
slug: fix-vscode-cpptools-cpu-usage-too-high
date: 2024-12-04T18:27:07+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
  - VS Code 插件
  - C/C++
  - cpptools
categories:
  - 经验分享
---

# VS Code C/C++ 插件 cpptools CPU 占用过高

在使用 VS Code 打开包含大量文件的 C/C++ 项目时，经常会遇到 CPU 占用过高的问题。尤其是当同时打开多个 VS Code 窗口时，CPU 占用会成倍增加，有时甚至高达 400%，几乎完全占满 4 个核心。经排查，这是由于 C/C++ 插件在分析代码，而我的项目中包含 Linux 源码、u-boot 源码、buildroot 源码等大型代码库，导致插件一直在进行代码分析，进而引发系统卡顿。

## 解决方法：排除目录

可以通过修改 settings.json 文件，为 C/C++ 插件排除特定目录。以下是我的配置示例：

```json
"C_Cpp.files.exclude": {
	"**/.git": true,
	"**/workdir": true,
	"**/wkdir": true,
	"**/build": true,
	"**/bin": true,
	"/PROJECT_ROOT/COMPONENT/ANONYMIZED": true,
}
```

- 值得注意的是，配置的是`C_Cpp.files.exclude`而非`files.exclude`，如果配置成后者，这些目录会被 VS Code 的文件资源管理器排除，跟消失了一样。

## 配置说明

`**/workdir` 和 `**/wkdir`：我的项目中，大量外部源码存储在这两个目录中。

`/PROJECT_ROOT/COMPONENT/ANONYMIZED`：该目录是 C/C++ 插件缓存代码分析结果的存储位置，排除它可以防止插件重复递归分析自身的缓存文件而产生死循环。

## 参考

有关插件缓存目录（`/PROJECT_ROOT/COMPONENT/ANONYMIZED`）的更多讨论，可以参考 [GitHub Issue](https://github.com/microsoft/vscode-cpptools/issues/10271#issuecomment-1363489906)。