---
title: "在 VMware Fusion 中扩展 Ubuntu 虚拟机磁盘容量"
slug: expand-ubuntu-disk-capacity
description: 
date: 2024-10-19T16:18:16+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags: 
    - VMware Fusion
    - Ubuntu
categories:
    - 经验分享
---

# 在 VMware Fusion 中扩展 Ubuntu 虚拟机磁盘容量

在使用 VMware Fusion 时，可能会遇到 Ubuntu 虚拟机磁盘空间不足的问题。本文将介绍如何在 VMware Fusion 中扩展 Ubuntu 虚拟机的磁盘容量，以确保系统能够正常运行并满足更多存储需求。

## 在 VMware Fusion 中增加 Ubuntu 的硬盘分配空间

首先将 Ubuntu 虚拟机关机，然后在 VMware Fusion 中配置硬盘空间。

![硬盘配置](post/expand-ubuntu-disk-capacity/imgs/image.png)

## 在 Ubuntu 中分配新增空间

此时新增的空间只是未分配空间，并没有真正用于 Ubuntu，磁盘大小并没有改变。

可以使用 GParted 图形化分区编辑工具对硬盘进行分配。

![GParted 工具](post/expand-ubuntu-disk-capacity/imgs/image-1.png)

如图所示，新增的 130G 空间尚未分配。

![未分配空间](post/expand-ubuntu-disk-capacity/imgs/image-2.png)

首先点击原有的磁盘空间，再点击 resize 按钮，将其扩大至最大。

![调整磁盘大小](post/expand-ubuntu-disk-capacity/imgs/image-3.png)

完成后可以看到根目录磁盘空间已经扩大至 150G。

![扩展后的磁盘空间](post/expand-ubuntu-disk-capacity/imgs/image-4.png)