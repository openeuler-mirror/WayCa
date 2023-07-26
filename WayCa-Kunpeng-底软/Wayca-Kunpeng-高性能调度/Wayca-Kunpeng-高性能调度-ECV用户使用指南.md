# Wayca-Kunpeng-高性能调度-ECV用户使用指南

当前Linux主线已支持该特性

## 1. 介绍

armv8.7引入ECV特性来读取系统用计数器，以减小为了确保指令执行顺序而设置isb(内存屏障)带来的开销。ECV提供了一组与通用寄存器对照的
自同步寄存器，可读取这组映射通用计数器值的寄存器从而减少isb带来的开销。

## 2. 内存屏障

arm架构属于弱内存序，执行指令时不一定会顺序执行，可能会对指令进行重排序。
内存屏障的引入，本质上是由于CPU重排序指令引起的，主要源自以下几种场景：
- 编译器编译时的优化
- 处理器执行时的多发射和乱序优化
- 读取和存储指令的优化
- 缓存同步顺序（导致可见性问题）

内存屏障就是一类同步屏障指令，是CPU或者编译器在对内存随机访问的操作中的一个同步点，只有在此点之前的所有读写操作都执行后才可以执行
此点之后的操作。

## 3. 相关寄存器

CNTPCT_EL0：保存64位物理计数值，该寄存器的值映射到AARCH32的通用计数器CNTPCT。
CNTPCTSS_EL0：以自同步方式保持物理计数值（与CNTPCT值相同），此寄存器只有FEAT_ECV使能时才能访问，且不能被投机访问。CNTPCTSS_EL0
的自同步以及不能被投机性访问的性质保证了CPU不能对对该寄存器的读写操作重排序。
CNTPOFF_EL2：保存 64 位物理偏移量。这是启用增强计数器虚拟化时 AArch64 物理计时器和计数器的偏移量。当在当前安全状态下实现并启用
EL2 时，如果满足以下任一条件，物理计数器将使用固定物理偏移量零：
- CNTHCTL_EL2.ECV为 0
- SCR_EL3.ECVEn为 0
- HCR_EL2 .{E2H, TGE} 为{1, 1}

相较于从EL2或EL3访问CNTPCTESS_EL0时，EL0和EL1该寄存器的值为(PhysicalCountInt<63:0> - CNTPOFF_EL2 <63:0>)，其中
PhysicalCountInt<63:0> 是从 EL2 或 EL3 读取 CNTPCT_EL0 时返回的物理计数。

## 4. 内核配置支持

内核中没有ECV特性相关CONFIG，ARM64_HAS_ECV宏对ECV特性进行控制。

## 5. 编译器支持

ECV使能需要编译器支持

- GCC

当使用GCC编译器时, 需要支持以下选项

-march=armv8.7-a或-march=Armv8-a+ecv

- Clang

当使用Clang编译器时, 需要支持-march=ecv选项

## 6. 软件接口

用户可以通过/proc/cpuinfo或HWCAP或者查询ID_AA64MMFR0_EL1.ECV域查询硬件对于ECV的支持情况。

## 7. 涉及代码和使能

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
