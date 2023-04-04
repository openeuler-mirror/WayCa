# Wayca-Kunpeng-高性能调度-BTI用户使用指南

当前Linux主线已支持该特性

## 1. 介绍

arm64架构是绝大多数移动设备的核心，这也就意味着arm64设备会成为全球攻击者的一个关注目标。因此，大家越来越关注于一些对arm64系统进行加固的技术。BTI是其中一种方法，这个功能是用来捕捉野跳转的。理念很简单：打开BTI的情况下，每一个间接跳转之后碰到的第一条指令必须是一个特殊的BTI指令。这个指令在不具备BTI的系统上就是no-op（什么都不做的指令）；在具有BTI机制的硬件上，BTI指令可以不报出fault。如果跳过去的代码开头不是BTI指令，则会马上把当前进程杀死。

## 2. 内核相关配置

BTI使能需要开启以下内核配置：
- CONFIG_ARM64_PTR_AUTH
- CONFIG_ARM64_BTI
- CONFIG_ARM64_BTI_KERNEL (依赖CC_HAS_BRANCH_PROT_PAC_RET_BTI)

需要注意的是BTI依赖编译器支持, 要求GCC版本大于等于9, 或Clang版本大于等于8. 要求编译器支持-mbranch-protection=pac-ret+leaf+bit.

## 3. 软件接口

CPU通过ID_AA64PFR1_EL1描述BTI支持情况：当BIT[3:0]为0b0001时, 硬件支持BTI; 0b0000时硬件不支持。用户可以通过读取该寄存器获取硬件支持情况。

BTI主要由内核及编译器支持, 不需要用户态进行配置. 内核通过sctlr_el1控制BTI硬件使能, 通过PSTATE.BTYPE(Branch Target Identification Bit)判断是否跳转异常.

## 4. 涉及代码与使能
| COMMITID | SUBJECT | openeuler OLK-5.10 enabled（Y/N）|
| ---------- | -------- | ---------- |
| ab7876a98a21 | arm64: elf: Enable BTI at exec based on ELF program properties | Y |
| ec94a46ee7ac | arm64: BTI: Decode BYTPE bits when printing PSTATE | Y |
| 0537c4cd71e3 | arm64: BTI: Reset BTYPE when skipping emulated instructions | Y |
| 30685d789c48 | KVM: arm64: BTI: Reset BTYPE when skipping emulated instructions | Y |
| 383499f8863e | arm64: BTI: Add Kconfig entry for userspace BTI |Y|
| 5d1b631c773f | arm64: bti: Document behaviour for dynamically linked binaries |Y|
| 47d67e4d1918 | arm64: insn: Report PAC and BTI instructions as skippable |Y|
| 92e2294d870b | arm64: bti: Support building kernel C code using BTI |Y|
| 714a8d02ca4d | arm64: asm: Override SYM_FUNC_START when building the kernel with BTI |Y|
| c8027285e366 | arm64: Set GP bit in kernel page tables to enable BTI for the kernel |Y|
| fa76cfe65c1d | arm64: bpf: Annotate JITed code for BTI |Y|
| 97fed779f2a6 | arm64: bti: Provide Kconfig for kernel mode BTI |Y|
| 3a9b136c998f | arm64: asm: Provide a mechanism for generating ELF note for BTI |Y|
| a6aadc28278a | arm64: vdso: Annotate for BTI |Y|
| 5e02a1887fce | arm64: vdso: Force the vDSO to be linked as BTI when built for BTI |Y|
| bf740a905ffe | arm64: vdso: Map the vDSO text with guarded pages when built for BTI |Y|
| 3a88d7c5c944 |  arm64: kconfig: Update and comment GCC version check for kernel BTI |Y|
| e4e9f6dfeedc | arm64: bti: Fix support for userspace only BTI |Y|
| 9a964285572b | arm64: vdso: Don't prefix sigreturn trampoline with a BTI C instruction |Y|
| b9249cba25a5 | arm64: bti: Require clang >= 10.0.1 for in-kernel BTI support |Y|
| 2d21889f8b5c | arm64: Don't insert a BTI instruction at inner labels |Y|
| 2980e6070eef | Revert "arm64: bti: Require clang >= 10.0.1 for in-kernel BTI support" | Y |
