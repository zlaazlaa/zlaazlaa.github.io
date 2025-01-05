---
title: "使用 lite-cornea 调试 ARM Fixed Virtual Platforms (FVPs)"
description:
slug: debug-fvp-via-gdb
date: 2025-01-05T15:59:16+08:00
image:
math:
license:
hidden: false
comments: true
draft: flase
tags:
  - ARM Fixed Virtual Platforms (FVPs)
  - debug
categories:
  - 经验分享
---

## 概述

在调试 ARM FVP 模型时，直接通过 GDB 连接是行不通的。虽然 ARM 官方为付费版 Fast Model 提供了 [GDB 插件](https://developer.arm.com/documentation/100964/1115/Plug-ins-for-Fast-Models/GDBRemoteConnection)，但免费版 Base Model 并不支持该功能。取而代之，Base Model 提供了一种名为 Iris Debug Server 的 Python Debug API。因此，要使用 GDB 调试 Base FVP，需要通过中间工具进行协议转译。

本文将介绍如何使用 lite-cornea 工具简化 FVP 的调试过程。

## 调试工具选择

目前找到的 Iris-to-GDB 转译工具有以下两个：

- **[lite-cornea](https://github.com/Linaro/lite-cornea.git)**
  - 基于 Rust，可直接嵌入 GDB 终端，命令兼容性较好。
- **[iris-gdb-wrapper](https://github.com/santongding/iris-gdb-wrapper)**
  - 基于 Python，需要先运行后端服务，然后以远程模式连接 GDB。

经测试，lite-cornea 支持更多 GDB 命令（如 `dump`、`restore`），推荐使用。iris-gdb-wrapper 在使用这些命令时有一些 bug，不能正常导入导出内存。

## 安装 lite-cornea

### 安装 Rust 环境

运行以下命令快速安装：

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

安装完成后，**重启终端**并验证 Rust：

```shell
cargo --version
```

### 编译并安装 lite-cornea

```shell
git clone https://github.com/Linaro/lite-cornea.git
cd lite-cornea
cargo build
sudo cp target/debug/cornea /usr/bin/
cornea --help
```

## 使用 GDB + lite-cornea 调试 FVP

### 配置 FVP 启动参数

在启动 FVP 时，添加 -I 参数以启用 Iris Debug Server。

### 编写 GDB 调试脚本

创建一个 GDB 启动脚本（例如 debug.sh）并添加以下内容：

```shell
#!/usr/bin/env -S gdb -q -ix
set architecture aarch64
add-symbol-file /path/to/your/symbol/file
target remote | cornea gdb-proxy component.FVP_Base_RevC_2xAEMvA.cluster0.cpu0
```

### 启动调试

运行 FVP 后，执行调试脚本：

```shell
sh debug.sh
```

脚本执行成功后，会自动进入 GDB 调试终端，默认停在 0x0 的位置，方便后续调试：

```shell
$ ./gdb-connect-fvp2.sh
The target architecture is set to "aarch64".

warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000000000 in ?? ()
(gdb)
```

**注意**：

cornea 参数中的 FVP_Base_RevC_2xAEMvA 需替换为实际运行的 FVP 模型名称，与 FVP 的二进制文件名（例如 FVP_Base_RevC-2xAEMvA）相似。
如果不确定模型名称，请参考 [ARM FVP 文档](https://documentation-service.arm.com/static/615eda5ce4f35d24846799a7) 中的模型目录：

![ARM FVP Base Model目录](post/debug-fvp-via-gdb/imgs/fvp-guide-catalog.png)
