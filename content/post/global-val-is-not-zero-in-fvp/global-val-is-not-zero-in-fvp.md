---
title: "ARM Fixed Virtual Platforms (FVPs)中运行 Hypervisor，Guest OS 的全局变量没有置零"
description: 
date: 2025-01-06T21:01:07+08:00
slug: global-val-is-not-zero-in-fvp
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
  - ARM Fixed Virtual Platforms (FVPs)
  - ELF
categories:
  - 经验分享
---

## 前言

最近在为 Hypervisor 适配新平台 ARM Fixed Virtual Platforms (FVPs)，具体架构是 FVP + Hypervisor + Guest OS，发现 Guest OS 无法启动。然而，QEMU + Hypervisor + Guest OS 一切正常。进一步调试后发现，在 FVP 场景下，Guest OS 中所有未初始化的全局变量均不是零值，导致默认它们为零的代码出现错误。

根据 C 标准，未初始化的全局变量应默认置零：

> **如果一个具有自动存储周期的对象不被显式地初始化，那么其值是不确定的。如果具有静态或线程存储周期的一个对象不被显式初始化，那么**
>
> —— 如果它具有指针类型，那么它被初始化为一个空指针；
>
> —— 如果它具有算术类型，那么它被初始化为（正的或负的）零；
>
> —— 如果它是一个聚合类型，那么每个成员被初始化（递归地），根据以上规则，并且任何填充被初始化为零比特；
>
> —— 如果它是一个联合体，那么第一个命名成员被（递归地）初始化，根据以上规则，并且填充被初始化为零比特。

## 问题分析

对于 ELF 文件来说，存储变量的叫数据段。数据段分为 .data 段和 .bss 段，其中 .data 段存储**已经初始化**的全局变量和静态变量，.bss 段存储**未经初始化**的全局变量。

.bss 段属于 nload 段，在镜像中不占空间，镜像被加载后，才会映射到内存，此时 .bss 段的内容继承自被映射的内存区域。

很不幸，我运行的 Guest OS 并没有清零自己的 .bss 段，而且 Hypervisor 的实现也比较简单，仅仅是做了以下内存映射：GPA -> HVA -> HPA，将自身的 memory 区域一一映射给 Guest OS，也没有对这部分内存做置零处理。

但是为什么 QEMU + Hypervisor + Guest OS 正常运行，而 FVP + Hypervisor + Guest OS 不行呢？

这是因为 QEMU 提供的可用内存的默认值都是 0：
![qemu-memory-in-gdb](post/global-val-is-not-zero-in-fvp/imgs/qemu-memory-in-gdb.png)
而 FVP 提供的可用内存区域默认值是 0xcfdfdfdf 与 0xdfdfdfcf 的循环：
![fvp-memory-in-gdb](post/global-val-is-not-zero-in-fvp/imgs/fvp-memory-in-gdb.png)

---
具体见 FVP 的 params：

```shell
FVP_Base_RevC-2xAEMvA --list-params | grep fill
bp.dmc.fill1=3755990991                               # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dmc.fill2=3487555551                               # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dmc_phy.fill1=3755990991                           # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dmc_phy.fill2=3487555551                           # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dummy_local_dap_rom.fill1=3755990991               # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dummy_local_dap_rom.fill2=3487555551               # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dummy_ram.fill1=3755990991                         # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dummy_ram.fill2=3487555551                         # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dummy_usb.fill1=3755990991                         # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.dummy_usb.fill2=3487555551                         # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.ns_dram.fill1=3755990991                           # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.ns_dram.fill2=3487555551                           # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.psram.fill1=3755990991                             # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.psram.fill2=3487555551                             # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.rl_dram.fill1=3755990991                           # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.rl_dram.fill2=3487555551                           # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.rt_dram.fill1=3755990991                           # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.rt_dram.fill2=3487555551                           # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.s_dram.fill1=3755990991                            # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.s_dram.fill2=3487555551                            # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.secureDRAM.fill1=3755990991                        # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.secureDRAM.fill2=3487555551                        # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.secureSRAM.fill1=3755990991                        # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.secureSRAM.fill2=3487555551                        # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.sram.fill1=3755990991                              # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.sram.fill2=3487555551                              # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.vram.fill1=3755990991                              # (int   , init-time) default = '0xdfdfdfcf' : Fill pattern 1, initialise memory at start of simulation with alternating fill1, fill2 pattern
bp.vram.fill2=3487555551                              # (int   , init-time) default = '0xcfdfdfdf' : Fill pattern 2, initialise memory at start of simulation with alternating fill1, fill2 pattern
cluster0.ext_abort_fill_data=18302063723721653757     # (int   , init-time) default = '0xfdfdfdfcfcfdfdfd' : Returned data, if external aborts are asynchronous
cluster0.randomize_unknowns_at_reset=0                # (bool  , init-time) default = '0'      : Will fill in unknown bits in registers at reset with random value using register_reset_data as seed, it overrides scramble_unknowns_at_reset
cluster0.register_reset_data=0                        # (int   , init-time) default = '0x0'    : Data used to fill register bits when they become UNKNOWN at reset.
cluster0.register_reset_data_hi=0                     # (int   , init-time) default = '0x0'    : Data used to fill the upper-half of 128-bit registers when the bits become UNKNOWN at reset.
cluster0.scramble_unknowns_at_reset=1                 # (bool  , init-time) default = '1'      : Will fill in unknown bits in registers at reset with register_reset_data
cluster0.unpred_extdbg_unknown_bits=0                 # (int   , init-time) default = '0x0'    : Data used to fill only in UNKNOWN bit-fields of external debug registers e.g., EDPFR and EDDFR.
cluster1.ext_abort_fill_data=18302063723721653757     # (int   , init-time) default = '0xfdfdfdfcfcfdfdfd' : Returned data, if external aborts are asynchronous
cluster1.randomize_unknowns_at_reset=0                # (bool  , init-time) default = '0'      : Will fill in unknown bits in registers at reset with random value using register_reset_data as seed, it overrides scramble_unknowns_at_reset
cluster1.register_reset_data=0                        # (int   , init-time) default = '0x0'    : Data used to fill register bits when they become UNKNOWN at reset.
cluster1.register_reset_data_hi=0                     # (int   , init-time) default = '0x0'    : Data used to fill the upper-half of 128-bit registers when the bits become UNKNOWN at reset.
cluster1.scramble_unknowns_at_reset=1                 # (bool  , init-time) default = '1'      : Will fill in unknown bits in registers at reset with register_reset_data
cluster1.unpred_extdbg_unknown_bits=0                 # (int   , init-time) default = '0x0'    : Data used to fill only in UNKNOWN bit-fields of external debug registers e.g., EDPFR and EDDFR.
```

以上配置项显示了 FVP 可以对如下内存区域进行配置：

- **`bp.dmc.*` 和 `bp.dmc_phy.*`**  
  表示 DRAM Controller（内存控制器）及其物理层相关的内存填充值。  
  用途：初始化内存控制器的地址空间，可能包括配置寄存器、缓冲区等。

- **`bp.ns_dram.*`**  
  表示 Non-Secure DRAM（非安全动态随机存取存储器）的填充值。  
  用途：初始化普通内存区域（非安全内存），模拟器启动时填充的默认模式。

- **`bp.rl_dram.*` 和 `bp.rt_dram.*`**  
  表示 Real-time Left 和 Real-time Right DRAM，分别对应系统的左右两部分实时内存。  
  用途：通常用于模拟分区的实时内存。

- **`bp.s_dram.*` 和 `bp.secureDRAM.*`**  
  表示 Secure DRAM（安全动态随机存取存储器）的填充值。  
  用途：存储安全相关数据，例如密钥、加密缓存等。

- **`bp.psram.*`**  
  表示 PSRAM（伪静态随机存取存储器）的填充值。  
  用途：嵌入式设备中用于提高性能的混合型存储器。

- **`bp.sram.*` 和 `bp.secureSRAM.*`**  
  表示 SRAM（静态随机存取存储器）和 Secure SRAM 的填充值。  
  用途：SRAM 通常用于高速缓存或快速存储，Secure SRAM 用于安全用途。

- **`bp.vram.*`**  
  表示 VRAM（视频随机存取存储器）的填充值。  
  用途：用于存储图形数据或显示缓冲区。

- **`bp.dummy_*`**  
  表示模拟器中用于占位或辅助调试的虚拟内存区域（如 dummy_ram、dummy_usb）。  
  用途：不对应真实硬件，而是提供测试或开发的功能。

---

关于 fill1 和 fill2，如果他们的值分别是 A 和 B，那么 FVP 的对应内存区间的值就会是 ABABABABABABABABABAB...... 的循环。

如果不对 FVP 内存初始值进行配置，.bss 段“展开”后的内容就是默认的 0xcfdfdfdfdfdfdfcf，这也是为什么 debug 过程中看到所有未经初始化的值都是 0xcfdfdfdfdfdfdfcf。

对于我这个例子，Guest OS 镜像是嵌入 Hypervisor 镜像中的，经过测试，发现 Hypervisor 镜像所在内存区间为 bp.s_dram，为 FVP 添加如下启动参数后，Guest OS 可以正常启动：

```shell
	-C bp.s_dram.fill1=0x0 \
	-C bp.s_dram.fill2=0x0
```

PS:

关于为什么是 bp.s_dram，[FVP 手册](https://documentation-service.arm.com/static/5f4d1264ca7b6a3399375cb4)中可以找到一些蛛丝马迹。~~其实我是挨个试的~~

Hypervisor 镜像所在内存区间的安全属性是 `P`
![](post/global-val-is-not-zero-in-fvp/imgs/fvp-memory-map.png)

对于`P`，默认的处理策略是安全访问和非安全访问全部拒绝，我猜测拒绝非安全访问的区间，都在 bp.s_dram 的配置范围内。但是我也查看了一些`N/S`区间，并不是所有的`N/S`区间都在 	bs.s_dram 的配置范围内。如果有知道的同学可以在评论区⬇️讨论下。
![](post/global-val-is-not-zero-in-fvp/imgs/fvp-secure-memory.png)
