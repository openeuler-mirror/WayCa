# Wayca-Kunpeng-高性能调度-SVE用户使用指南

当前Linux主线与Openeuler OLK-5.10已支持该特性

## 1. 介绍

随着 Neon 架构扩展（其指令集具有固定的 128 位向量长度）的开发，Arm 设计了可扩展向量扩展 (SVE) 作为 AArch64 的下一代 SIMD 扩展。SVE引入可扩展概念， 允许灵活的向量长度实现，使其能够在现在或将来的多应用场景下实现伸缩，允许CPU设计者自由选择向量的长度来实现。矢量长度可以从最小 128 位到最大 2048 位不等，以 128 位为增量。在当前实现中，最大实现向量长度为256位。SVE的设计保证同样的应用程序可以在支持SVE的不同实现上执行，而无需重新编译代码。在高性能计算跟机器学习场景具有优势,适用于大量数据处理的场景，通过将数据向量化，指令对向量进行操作，从而实现指令对数据批量处理，以加快运算速度。SVE2 扩展了 SVE 指令集，以支持高处理性能和机器学习之外的数据处理领域，例如计算机视觉、多媒体、游戏、LTE 基带处理和通用软件。

sve添加了以下寄存器：
- 32个可扩展的向量寄存器，Z0-Z31：
![avatar](https://documentation-service.arm.com/static/63b8334140f3173eeee2a7ca?token=)
- 16个可扩展的谓词寄存器，P0-P15：
![avatar](https://documentation-service.arm.com/static/63b8334140f3173eeee2a7cd?token=)
- 一个First Fault 谓词寄存器（FFR）：
可扩展的向量系统控制寄存器ZCR_Elx

## 2. 内核相关配置

SVE使能需要开启以下内核配置：
- CONFIG_ARM64_SVE

## 3. 软件接口

### 3.1 支持检索

Linux下SVE支持情况通过CPU信息文件（/proc/cpuinfo）传递给用户态，也可以通过读取HWCAP_SVE或ID_AA64PFR0_EL1寄存器获取硬件SVE支持情况。

### 3.2  SVE配置

- 系统默认SVE向量长度配置
Linux支持通过/proc/sys/abi/sve_default_vector_length文件获取并设置系统默认的SVE向量长度。除非用户通过prctl对线程SVE向量长度进行配置，默认情况下均使用该配置作为线程的默认长度。
- 进程粒度的向量长度配置
Linux支持运行时动态配置当前进程使用的SVE向量长度，可以通过prctl系统调用对当前进程的SVE向量寄存器宽度进行配置和管理，当前支持以下两个命令：
```
PR_SVE_SET_VL
        设置当前线程的向量长度。当PR_SVE_VL_INHERIT标志设置时，子线程会继承当前的
        VL设置；否则子线程使用系统默认的向量长度设置。当PR_SVE_SET_VL_ONEXEC标志
        设置时，仅当当前线程执行execve()时向量长度修改才会生效。
        当修改生效时，P0-P15，FFR以及Z0-Z31寄存器（非低128bits）的数据均未定义。
PR_SVE_GET_VL
        获取当前线程的向量长度设置。
```

## 4. 涉及代码和使能

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

## 5.测试用例

以数组加权相加功能函数为例，一个为非sve版本函数，一个为sve版本函数
void daxpy_1_1_no_sve(int64_t n, double da, double *dx, double *dy)
{
        for (int64_t i = 0; i < n; ++i) {
                dy[i] = dx[i] * da + dy[i];
        }
}

include <arm_sve.h>
void daxpy_1_1(int64_t n, double da, double *dx, double *dy)
{
        int64_t i = 0;
        svbool_t pg = svwhilelt_b64(i, n);
        do
                {
                        svfloat64_t dx_vec = svld1(pg, &dx[i]);
                        svfloat64_t dy_vec = svld1(pg, &dy[i]);
                        svst1(pg, &dy[i], svmla_x(pg, dy_vec, dx_vec, da));
                        i += svcntd();
                        pg = svwhilelt_b64(i, n);
                }
        while (svptest_any(svptrue_b64(), pg));
}

使用gcc -march=armv8-a+sve xxx.c  xxx命令在不支持sve系统与支持sve系统上分别运行./xxx运行
