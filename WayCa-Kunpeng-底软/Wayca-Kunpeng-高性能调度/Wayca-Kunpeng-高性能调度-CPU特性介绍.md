# Wayca-Kunpeng-高性能调度-CPU特性介绍

## 特性详解

### 特性1：SVE(Scalable Vector Extension)可扩展矢量扩展

- 特性详解

随着 Neon 架构扩展（其指令集具有固定的 128 位向量长度）的开发，Arm 设计了可扩展向量扩展 (SVE) 作为 AArch64 的下一代 SIMD 扩展。SVE引入可扩展概念， 允许灵活的向量长度实现，使其能够在现在或将来的多应用场景下实现伸缩，允许CPU设计者自由选择向量的长度来实现。矢量长度可以从最小 128 位到最大 2048 位不等，以 128 位为增量。在当前实现中，最大实现向量长度为256位。

- 源码仓库： https://gitee.com/openeuler/kernel/
- 特性代码： arch/arm64/kernel
- 支持版本： openEuler 22.03 lts、openEuler 22.03 lts sp1
- 回合的关键patches:

| COMMITID     | SUBJECT      |openeuler OLK-5.10 enabled（Y/N）|
| ---------- | ---------- | -----------|
| 27e64b4be4b8 | regset: Add support for dynamically sized regsets |Y|
| 94ef7ecbdf6f | arm64: fpsimd: Correctly annotate exception helpers called from asm | Y |
| abf73988a7c2| arm64: signal: Verify extra data is user-readable in sys_rt_sigreturn |Y|
| 93390c0a1b20| arm64: KVM: Hide unsupported AArch64 CPU features from guests |Y|
| b472db6cf8c6 | arm64: efi: Add missing Kconfig dependency on KERNEL_MODE_NEON|Y|
| 38b9aeb32fa7| arm64: Port deprecated instruction emulation to new sysctl interface |Y|
| 9cf5b54fafed| arm64: fpsimd: Simplify uses of {set, clear}_ti_thread_flag()|Y|
| 672365649cca |arm64/sve: System register and exception syndrome definitions |Y|
| 1fc5dce78ad1| arm64/sve: Low-level SVE architectural state manipulation functions |Y|
| ddd25ad1fde8| arm64/sve: Kconfig update and conditional compilation support |Y|
| d0b8cd318788| arm64/sve: Signal frame and context structure definition |Y|
| 22043a3c082a | arm64/sve: Low-level CPU setup |Y|
| bc0ee4760364 | arm64/sve: Core task context handling|Y|
| 79ab007c75d6| arm64/sve: Support vector length resetting for new processes |Y|
| 8cd969d28fd2 | arm64/sve: Signal handling support |Y|
| 7582e22038a2| arm64/sve: Backend logic for setting the vector length |Y|
| 8f1eec57cdcc  | arm64: cpufeature: Move sys_caps_initialised declarations |Y|
| 2e0f2478ea37 | arm64/sve: Probe SVE capabilities and usable vector lengths |Y|
| 1bd3f93641ec| arm64/sve: Preserve SVE registers around kernel-mode NEON use |Y|
| fdfa976cae5c | arm64/sve: Preserve SVE registers around EFI runtime service calls     |Y|
| 43d4da2c45b2 | arm64/sve: ptrace and ELF coredump support |Y|
| 2d2123bc7c7f | rm64/sve: Add prctl controls for userspace vector length management    |Y|
| 4ffa09a939ab| arm64/sve: Add sysctl to set the default vector length for new processes|Y|
| 17eed27b02da  | arm64/sve: KVM: Prevent guests from using SVE                         |Y|
| aac45ffd1f8e | arm64/sve: KVM: Treat guest SVE use as undefined instruction execution |Y|
| 07d79fe7c223 | arm64/sve: KVM: Hide SVE from CPU features exposed to guests |Y|
| 43994d824e84 | arm64/sve: Detect SVE and activate runtime support |Y|
| ce6990813f15 | arm64/sve: Add documentation |Y|
| 94b07c1f8c39| arm64: signal: Report signal frame size to userspace via auxv |Y|

### 特性2：BTI(Branch Target Identification)分支目标识别

- 特性详解

BTI（branch target identification）是其中一种方法，这个功能是用来捕捉wild jump（野跳转）的。理念很简单：打开BTI的情况下，每一个间接跳转（indirect jump）之后碰到的第一条指令必须是一个特殊的BTI指令。这个指令在不具备BTI的系统上就是no-op（什么都不做的指令）；在具有BTI机制的硬件上，BTI指令可以不报出fault。如果跳过去的代码开头不是BTI指令，则会马上把当前进程杀死。

- 源码仓库： https://gitee.com/openeuler/kernel/

- 特性代码： arch/arm64/kernel arch/arm64/mm/mmu.c arch/arm64/net/bpf_jit_comp.c

- 支持版本： openEuler 22.03 lts、openEuler 22.03 lts SP1

- 回合的关键patches:
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

### 特性3：AMU(Activity Monitors Extension)

- 特性详解

处理器包括基于AMUv1体系结构的活动监控。它旨在用于系统管理，而性能监控则针对用户和调试应用程序。活动监视器为系统电源管理和持续监控提供了有用的信息。活动监视器在操作中是只读的,它们的配置仅限于实现的最高异常级别。

Armv8允许实现最多16个计数器, 每个计数器可编程或统计固定事件. 当前系统可支持4个可编程的辅助计数器。

- 源码仓库： https://gitee.com/openeuler/kernel/

- 特性代码： arch/arm64/kernel arch/arm64/include/asm

- 支持版本： openEuler 22.03 lts、openEuler 22.03 lts SP1

- 回合的关键patches:
	| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED (Y/N) |
| ---------- | -------- |--------|
|2c9d45b43c39|arm64: add support for the AMU extension v1|Y|
|87a1f063464a |arm64: trap to EL1 accesses to AMU counters from EL0|Y|
|4fcdf106a433 |arm64/kvm: disable access to AMU registers from kvm guests|Y|
|6abde90881a5 |Documentation: arm64: document support for the AMU extension|Y|
|d91589556b6a|docs: amu: supress some Sphinx warnings|Y|
|59bff30ad6ce |Documentation: arm64: fix amu.rst doc warnings|Y|
|a0eef4a8acbb|Documentation: Chinese translation of Documentation/arm64/amu.rst|Y|
|ed159f972408|docs: zh_CN: amu.rst: fix document title markup|Y|

### 特性4：LSE(Large System Extensions)

- 特性详解

Armv8.1-A体系结构引入了新的原子指令，原子是加载的替代方案，存储以前体系结构版本中
使用的独占序列。Arm体系结构参考手册将原子指令称为Large System Extensions (LSE)。

LSE引入了一组原子指令：

- 比较和交换指令, CAS and CASP
- 原子内存操作指令, LD<OP> and ST<OP>, where <OP> is one of ADD, CLR, EOR, SET,
  SMAX, SMIN, UMAX, and UMIN
- 交换指令, SWP

LSE仅在A64中支持, 要求在Armv8.1及之后的版本必须实现。

- 源码仓库： https://gitee.com/openeuler/kernel/

- 特性代码： arch/arm64/kernel arch/arm64/include/asm

- 支持版本： openEuler 22.03 lts、openEuler 22.03 lts SP1

- 回合的关键patches:
	| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED (Y/N) |
| ---------- | -------- |--------|
| 9511ca19dafb | arm64: rwlocks: don't fail trylock purely due to contention | Y |
| 144e9697a9e7 | arm64: cpufeature.h: add missing #include of kernel.h | Y |
| c275f76bb4ce | arm64: atomics: move ll/sc atomics into separate header file | Y |
| 40a1db2434a1 | arm64: elf: advertise 8.1 atomic instructions as new hwcap | Y |
| d964b7229e7f | arm64: alternatives: add cpu feature for lse atomics | Y |
| c0385b24af15 | arm64: introduce CONFIG_ARM64_LSE_ATOMICS as fallback to ll/sc atomics | Y |
| c09d6a04d17d | arm64: atomics: patch in lse instructions when supported by the CPU | Y |
| 81bb5c642063 | arm64: locks: patch in lse instructions when supported by the CPU | Y |
| 084f903727e1 | arm64: bitops: patch in lse instructions when supported by the CPU | Y |
| c8366ba0fb65 | arm64: xchg: patch in lse instructions when supported by the CPU | Y |
| c342f78217e8 | arm64: cmpxchg: patch in lse instructions when supported by the CPU | Y |
| e9a4b795652f | arm64: cmpxchg_dbl: patch in lse instructions when supported by the CPU | Y |
| 0bc671d3f4be | arm64: cmpxchg: avoid "cc" clobber in ll/sc routines | Y |
| 4e39715f4b5c | arm64: cmpxchg: avoid memory barrier on comparison failure | Y |
| a82e62382fcb | arm64: atomics: tidy up common atomic{,64}_* macros | Y |
| 0ea366f5e1b6 | arm64: atomics: prefetch the destination word for write prior to stxr | Y |
| 6059a7b6e818 | arm64: atomics: implement atomic{,64}_cmpxchg using cmpxchg | Y |
| db26217e6f54 | arm64: atomic64_dec_if_positive: fix incorrect branch condition | Y |
| 95eff6b27c40 | arm64: kconfig: select HAVE_CMPXCHG_LOCAL | Y |

### 特性5：ECV(Enhanced Counter Virtualization)

- 特性详解

armv8.7引入ECV特性来读取系统用计数器，以减小为了确保指令执行顺序而设置isb(内存屏障)带来的开销。ECV提供了一组与通用寄存器对照的
自同步寄存器，可读取这组映射通用计数器值的寄存器从而减少isb带来的开销。

- 源码仓库： https://gitee.com/openeuler/kernel/

- 特性代码： arch/arm64/kernel arch/arm64/include/asm drivers/clocksource/

- 支持版本： openEuler 22.03 lts、openEuler 22.03 lts SP1

- 回合的关键patches:
| COMMITID | SUBJECT | openeuler OLK-5.10 enabled（Y/N） |
| ---------- | ---------- | ----------- |
| 57f27666f91a8 | clocksource/arm_arch_timer: Drop use of static key in arch_timer_reg_read_stable | Y |
| 68b0fd268c118 | arm64: Add CNT{P,V}CTSS_EL0 alternatives to cnt{p,v}ct_el0 | Y |
| 6acc71ccac718 | arm64: arch_timer: Allows a CPU-specific erratum to only affect a subset of CPUs | Y |
| f31e98bfae1c8 | arm64: arch_timer: mark functions as __always_inline | Y |
| 0ea415390cd34 | clocksource/arm_arch_timer: Use arch_timer_read_counter to access stable counters | Y |
| 75a19a0202db2 | arm64: arch_timer: Ensure counter register reads occur with seqlock held | Y |
| 1aee5d7a8120c | arm64: move from arm_generic to arm_arch_timer | Y |
| 0583fe478a7d9 | ARM: convert arm/arm64 arch timer to use CLKSRC_OF init | Y |
| 4ad499c94264a | Documentation: Add ARM64 to kernel-parameters.rst | Y |

### 特性6：MPAM(Memory Partitioning And Monitoring Extension)

- 特性详解

MPAM（Memory Partitioning And Monitoring Extension）是Armv8.4体系结构引入的可选扩展。MPAM仅在AArch64下支持。MPAM扩展为内存系
统组件控件提供了一个框架，支持对组件的一个或多个资源进行分区。

MPAM软件包括两个部分：MPAM资源池的发现和初始化和Cache/memory资源的分区管理。MPAM当前通过Linux的resctrl（复用x86 RDT的对外接口）
进行控制。

Cache/memory带宽资源的分区管理部分承接用户态接口的输入输出和资源池之间的交互，该部分参考原x86 RDT的实现，并进行适当拓展，其中包
括：通过SMMU对IO增加partID标记接口，重新设计实现L2/L3 cdp机制，增加MPAM特有的L2/L3/MB优先级控制接口等。

当前Linux主线没有支持MPAM特性，openEuler实现了该特性，且当前仅为preview特性。关于MPAM在openEuler上支持及使能也可以参考
[openEuler社区仓库](https://gitee.com/openeuler/community)的MPAM文档sig/Kernel/mpam.md。

- 源码仓库： https://gitee.com/openeuler/kernel/

- 特性代码： arch/arm64/kernel/mpam

- 支持版本： openEuler 22.03 lts、openEuler 22.03 lts SP1

- 回合的关键patches:
| openeuler OLK5.10 COMMITID | SUBJECT |
| ---------- | -------- |
| 073090c4f9e4 | arm64/mpam: debug: print debug info when create mon_data |
| 004963a037e0 | arm64/mpam: add group partid/pmg to tasks show |
| 26d503ccaf97 | arm64/mpam: pass rdtgroup when create mon_data dir |
| c89d858c75ed | arm64/mpam: support monitor read |
| 7f42238eb878 | arm64/mpam: support monitor |
| 285671724491 | arm64/mpam: support num_partids/num_pmgs |
| ff75ea02b109 | arm64/mpam: add mpam extension runtime detection |
| 2d21a72d73fd | arm64/mpam: print mpam caps info when booting |
| 36162d6f7918 | arm64/mpam: disable MPAM_SYS_REG_DEBUG |
| 260d3f830a87 | arm64/mpam: support monitor |
| 32b8643fe582 | arm64/mpam: operation not permitted when remove a ctrl group with a mondata |
| 6910641acc13 | arm64/mpam: free mon when remove momgroups |
| 453c7520c6be | arm64/mpam: mon: add WARN_ON for debug free_pmg |
| 5c8245a49e5b | arm64/mpam: add num_monitors in info dir |
| 7ebf22416b68 | arm64/mpam: get num_mon & num_pmg from hardware |
| 82fea323c009 | arm64/mpam: don't reserve mon 0, we can use it as nomarl |
| 577782afdfbb | arm64/mpam: get alloc/mon capable/enabled from h/w |
| 7d1fba341864 | arm64/mpam: alloc/mon capable/enabled debug |
| 13fcfa015754 | arm64/mpam: add L3TALL & HHALL |
| 99f06cbe3464 | arm64/mpam: enable alloc/mon capable when MPAM enabled |
| b0b8538e197b | arm64/mpam: monitor pmg as a property of partid |
| 5bb872c2473d | arm64/mpam: fix HHA MAX SET/GET operation |
| fdb02ee44ed3 | arm64/mpam: don't allowd create mon_groups when out of mon/pmg |
| 7770d6e5a35a | arm64/mpam: use 5% as min memory bandwidth |
| 45f455df5bb5 | arm64/mpam: debug: remove debug pr_info at schemata |
| 145a91948fff | arm64/mpam: support L3TALL, HHALL |
| f2f34e16f22b | arm64/mpam: hard code mpam resource for Hi1620 2P |
| ed1d8ee9d757 | arm64/mpam: add cmdline option: mpam |
| 294eb2438565 | arm64/mpam: fix compile warning |
| cceab46d57d4 | mpam: Code security rectification |
| 593cba04174e | mpam: fix potential resource leak in mpam_domains_init |
| 74aab3fda289 | arm64/mpam: fix hard code address map for 1620 2P |
| fb55aed51c62 | arm64/mpam: destroy domain list when failed to init |
| fc426834088c | arm64/mpam: unmap all previous address when failed |
| 4bf3613e3b1a | arm64/mpam: only add new domain node to domain list |
| 0833601fa952 | arm64/mpam: remove unsupported resource |
| 8a74962dfb8a | arm64/mpam: update group flags only when enable sucsses |
| 143014f76c3e | arm64/mpam: get num_partids from system regs instead of hard code |
| 843a90dbbf20 | arm64/mpam: correct num of partid/pmg |
| 8e62aa8be825 | arm64/mpam: remove unnecessary debug message and dead code |
| 9525089d5fb5 | arm64/mpam: fix a missing unlock in error branch |
| aa65a72294f3 | arm64/mpam: cleanup debuging code |
| 59890b617805 | arm64/mpam: use snprintf instead of sprintf |
| 434eea4a4fc8 | mpam : fix missing fill MSMON_CFG_MON_SEL register |
| 8d83a69d9250 | mpam : fix monitor's disorder from |
| 1fef4872ac94 | arm64/mpam: cleanup the source file's licence |
| 5c3d89e3ae43 | ACPI 6.x: Add definitions for MPAM table |
| d43a6c0c538f | MPAM / ACPI: Refactoring MPAM init process and set MPAM ACPI as entrance |
| 88520d2383ac | arm64/mpam: Fix unreset resources when mkdir ctrl group or umount resctrl |
| dfa6e6512f5a | arm64/mpam: Supplement err tips in info/last_cmd_status |
| 1ce09eed3e96 | arm64/mpam: Preparing for MPAM refactoring |
| a2e55a9889e5 | arm64/mpam: Add mpam driver discovery phase and kbuild boiler plate |
