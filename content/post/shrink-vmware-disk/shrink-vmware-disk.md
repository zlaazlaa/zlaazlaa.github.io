---
title: "降低 VMware Fusion 的硬盘占用"
description: "介绍如何使用 VMware Tools 释放 VMware Fusion 虚拟机未使用的硬盘空间，减少宿主机硬盘占用。"
date: 2025-02-21T16:08:17+08:00
slug: shrink-vmware-disk
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
    - VMware Fusion
categories:
  - 经验分享
---

在使用 VMware Fusion 创建虚拟机时，分配给虚拟机的硬盘空间并不会立即占用宿主机的空间，而是根据虚拟机的实际需求动态申请。

经过长时间使用，VMware Fusion 占用宿主机的硬盘空间可能会大于虚拟机实际使用的空间。这是因为 VMware Fusion 只会向宿主机申请空间，而不会主动释放。此时，需要借助 VMware Tools 手动释放硬盘空间。

## 安装 VMware Tools

可以参考[官方文档](https://techdocs.broadcom.com/cn/zh-cn/vmware-cis/vsphere/tools/12-5-0/vmware-tools-administration-12-5-0/installing-vmware-tools/manually-install-vmware-tools-on-linux.html)，或者也可以使用 apt 安装：

```shell
sudo apt-get install open-vm-tools-desktop
```

## 使用 VMware Tools Shrink 硬盘

运行以下 shell 脚本，释放虚拟机未使用的磁盘空间：

```shell
#!/bin/bash
 
disk_list=`sudo vmware-toolbox-cmd disk list`
 
for disk in ${disk_list}
do
    sudo vmware-toolbox-cmd disk wipe ${disk}
done
 
sudo vmware-toolbox-cmd disk shrinkonly
```
