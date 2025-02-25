---
title: "ARM Fixed Virtual Platforms (FVPs) + ATF 多核启动"
description: "介绍 ARM FVP 上的 ATF 多核启动流程和常见配置，包含 PSCI 唤醒及 mailbox 机制"
date: 2025-02-25T14:10:15+08:00
slug: fvp-atf-multi-core-startup
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
  - ARM Fixed Virtual Platforms (FVPs)
  - ATF
categories:
  - 经验分享
---

## ATF 的两种多核启动流程

ATF 支持两种多核启动流程：

- SoC 启动时仅有一个核心上电
- SoC 启动时所有核心同时上电

这两种启动方式由以下宏定义控制：

```c
// atf-src/aarch64/bl1_entrypoint.S

func bl1_entrypoint
	/* ---------------------------------------------------------------------
	 * If the reset address is programmable then bl1_entrypoint() is
	 * executed only on the cold boot path. Therefore, we can skip the warm
	 * boot mailbox mechanism.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_init_sctlr=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\  // 注意这里
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=bl1_exceptions		\
		_pie_fixup_size=0
```

```c
// atf-src/include/arch/aarch64/el3_common_macros.S

	.if \_secondary_cold_boot
		/* -------------------------------------------------------------
			* Check if this is a primary or secondary CPU cold boot.
			* The primary CPU will set up the platform while the
			* secondaries are placed in a platform-specific state until the
			* primary CPU performs the necessary actions to bring them out
			* of that state and allows entry into the OS.
			* -------------------------------------------------------------
			*/
		bl	plat_is_my_cpu_primary
		cbnz	w0, do_primary_cold_boot

		/* This is a cold boot on a secondary CPU */
		bl	plat_secondary_cold_boot_setup
		/* plat_secondary_cold_boot_setup() is not supposed to return */
		bl	el3_panic

	do_primary_cold_boot:
	.endif /* _secondary_cold_boot */
```

### 单核上电

当编译 ATF 时，如果将 COLD_BOOT_SINGLE_CPU 设置为 1，则采用第一种方式：冷启动时只有一个核心运行，secondary_cold_boot 宏不会被编译，直接执行 do_primary_cold_boot。此时，从核需要通过内核发出 PSCI 请求，由 ATF 处理后才能启动。下图展示了内核调用 PSCI 并由 ATF 响应的过程：

![kernel-call-psci](post/fvp-atf-multi-core-startup/imgs/kernel-call-psci.png)

为 FVP 启动参数添加追踪插件（--plugin FVP-PATH/Base_RevC_AEMvA_pkg/plugins/Linux64_armv8l_GCC-9.3/TarmacTrace.so）后，可以观察到 u-boot 启动时仍然只有单核（cpu0）运行，其他核心只有在进入 kernel 后才被唤醒：

![fvp-tarmactrace](post/fvp-atf-multi-core-startup/imgs/fvp-tarmactrace.png)

### 多核上电

如果将 COLD_BOOT_SINGLE_CPU 设置为 0，则采用第二种方式：冷启动时所有核心同时上电，主核执行 do_primary_cold_boot，从核执行 plat_secondary_cold_boot_setup。在 plat_secondary_cold_boot_setup 函数中，从核会进入低功耗状态，等待 kernel 通过 PSCI 请求唤醒并写入启动地址，然后通过检查 mailbox 是否为非零值来完成唤醒过程：

```c
// atf-src/plat/arm/board/fvp/aarch64/fvp_helpers.S

	mov_imm	x0, PLAT_ARM_TRUSTED_MAILBOX_BASE
	/* Wait until the entrypoint gets populated */
poll_mailbox:
	ldr	x1, [x0]
	cbz	x1, 1f
	br	x1
1:
	wfe
	b	poll_mailbox
```

具体采用哪种启动方式取决于编译 ATF 时 COLD_BOOT_SINGLE_CPU 的配置值。在 ATF 的默认设置中，该值通常为 0（定义于 atf-src/make_helpers/defaults.mk），各平台可根据需要在各自的 platform.mk 中进行覆盖。在 FVP 的配置中，COLD_BOOT_SINGLE_CPU 也被设置为 0。

## FVP 配置启动时核心上电情况

FVP 还允许配置启动时为哪些核心上电，这与 COLD_BOOT_SINGLE_CPU 的不同取值组合，会产生不同的启动行为，如下表所示：

<table>
  <tr>
	<th></th>
	<th>pctl.startup=0.0.0.0（仅主核上电）</th>
	<th>pctl.startup=0.0.0.*（所有核心上电）</th>
  </tr>
  <tr>
	<td>COLD_BOOT_SINGLE_CPU=0</td>
	<td style="background-color: lightgreen;">✅ 正常启动</td>
	<td style="background-color: lightgreen;">✅ 正常启动</td>
  </tr>
  <tr>
	<td>COLD_BOOT_SINGLE_CPU=1</td>
	<td style="background-color: lightgreen;">✅ 正常启动</td>
	<td style="background-color: lightcoral;">❌ 无法启动</td>
  </tr>
</table>

当 COLD_BOOT_SINGLE_CPU=1 且 pctl.startup=0.0.0.\* 时，由于 do_primary_cold_boot 函数被多个核心重复执行，系统无法正常启动。  
而在 COLD_BOOT_SINGLE_CPU=0 且 pctl.startup=0.0.0.0 的情况下，系统可以通过 PSCI 接口唤醒尚未上电的次核：

![atf-reset-cpu1](post/fvp-atf-multi-core-startup/imgs/atf-reset-cpu1.png)
