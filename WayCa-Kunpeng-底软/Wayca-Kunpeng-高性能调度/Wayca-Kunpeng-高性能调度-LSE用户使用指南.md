# Wayca-Kunpeng-高性能调度-LSE用户使用指南

当前Linux主线与Openeuler OLK-5.10已支持该特性

## 1. LSE介绍

在编程时，如果多个处理器或线程访问共享数据，并且至少有一个正在写入时，操作必须是原
子的。这意味着数据访问必须被视为相对于其他处理器的单独操作，以避免数据竞争。

微处理器的设计是将 read-modify-write和memory-register exchange等序列视为单独操作，
即使它们创建了对内存的多次访问，这种硬件使编程更容易。

在LSE之前的体系结构版本中，read-modify-write序列使用加载独占和存储独占指令。递增共
享变量使用以下序列：
LDXR to read current count (load exclusive)
ADD to add one to the shared variable
STXR to attempt to store to memory (store exclusive)
CMP to check if the operation succeeded

由于原子访问使用多个指令，每个处理器都需要实现独占监视器。独占监视器是一个硬件状态
机，用于跟踪read-modify-write 序列并匹配加载和存储。如果处理器数量较少，则工作正
常。增加处理器数量加上增加缓存，使得很难保持公平性，因为彼此更接近的处理器有更好的
机会完成原子序列。

Armv8.1-A体系结构引入了新的原子指令，原子是加载的替代方案，存储以前体系结构版本中
使用的独占序列。Arm体系结构参考手册将原子指令称为Large System Extensions (LSE)。

对于LSE，原子指令在单个指令中提供不可中断的read-modify-write序列。原子指令可以对
指定的内存位置执行简单的算术或逻辑运算，并将更新后的值返回给处理器。程序员从原子指
令中受益，因为与在序列失败时周围有循环的指令序列相比，指定单个指令更容易。LSE提高
了具有许多处理器的系统中原子操作的性能。

LSE引入了一组原子指令：

- 比较和交换指令, CAS and CASP.
- 原子内存操作指令, LD<OP> and ST<OP>, where <OP> is one of ADD, CLR, EOR, SET,
  SMAX, SMIN, UMAX, and UMIN.
- 交换指令, SWP.

LSE仅在A64中支持, 要求在Armv8.1及之后的版本必须实现.

硬件通过ID_AA64ISAR0_EL1.Atomic (BITS[23:20])字段描述LSE的支持:

- 0b0000: LSE原子指令不支持
- 0b0010: 支持LDADD, LDCLR, LDEOR, LDSET, LDSMAX, LDSMIN, LDUMAX, LDUMIN, CAS,
          CASP, and SWP 指令实施

0b0010表示硬件支持LSE, 从Armv8.1开始, 仅0b0010为合法值.

## 2. 原子指令介绍

原子指令对一个内存位置执行原子读和写操作，使得体系结构保证在该指令定义的读和写之间，
另一个观察者不会对该内存位置进行修改。

### 2.1 原子内存操作

原子内存操作指令只支持一种寻址方式：base register only. 为了进行权限检查和观察点，
所有原子内存操作指令都被视为执行加载和存储。如果未实现LSE2，则LD<OP>和ST<OP>指令需
要自然对齐，而未对齐的地址会生成对齐错误。

这些指令提供了排序选项，这些选项映射到体系结构中使用的acquire和release定义。具有
release语义的原子指令与Store-Release指令在多副本原子性方面具有相同的规则。这些操
作映射到acquire和release定义，并分别计算为Load-Acquire和tore-Release操作。

对于LD<OP>指令，当源寄存器和目标寄存器相同时，如果指令生成同步数据中止，则源寄存器
将恢复到指令执行前保持的值。

ST<OP>指令和目标寄存器为WZR或XZR的LD<OP>指令不被视为在做用于DMB LD barrier的读取。

| Mnemonic | Instruction | See |
| ------ | ---------- | :----- |
| LDADD | Atomic add | LDADD, LDADDA, LDADDAL, LDADDL |
| LDADDB | Atomic add on byte         | LDADDB, LDADDAB, LDADDALB, LDADDLB |
| LDADDH | Atomic add on halfword | LDADDH, LDADDAH, LDADDALH, LDADDLH |
| LDCLR  | Atomic bit clear | LDCLR, LDCLRA, LDCLRAL, LDCLRL |
| LDCLRB | Atomic bit clear on byte | LDCLRB, LDCLRAB, LDCLRALB, LDCLRLB |
| LDCLRH | Atomic bit clear on halfword | LDCLRH, LDCLRAH, LDCLRALH, LDCLRLH |
| LDEOR | Atomic exclusive OR | LDEOR, LDEORA, LDEORAL, LDEORL |
| LDEORB | Atomic exclusive OR on byte | LDEORB, LDEORAB, LDEORALB, LDEORLB |
| LDEORH | Atomic exclusive OR on halfword | LDEORH, LDEORAH, LDEORALH, LDEORLH |
| LDSET | Atomic bit set | LDSET, LDSETA, LDSETAL, LDSETL |
| LDSETB | Atomic bit set on byte | LDSETB, LDSETAB, LDSETALB, LDSETLB |
| LDSETH | Atomic bit set on halfword | LDSETH, LDSETAH, LDSETALH, LDSETLH |
| LDMAX | Atomic signed maximum | LDSMAX, LDSMAXA, LDSMAXAL, LDSMAXL |
| LDMAXB | Atomic signed maximum on byte | LDSMAXB, LDSMAXAB, LDSMAXALB, LDSMAXLB |
| LDMAXH | Atomic signed maximum on halfword | LDSMAXH, LDSMAXAH, LDSMAXALH, LDSMAXLH |
| LDMIN | Atomic signed minimum | LDSMIN, LDSMINA, LDSMINAL, LDSMINL |
| LDMINB | Atomic signed minimum on byte | LDSMINB, LDSMINAB, LDSMINALB, LDSMINLB |
| LDMINH | Atomic signed minimum on halfword | LDSMINH, LDSMINAH, LDSMINALH, LDSMINLH |
| LDUMAX | Atomic unsigned maximum | LDUMAX, LDUMAXA, LDUMAXAL, LDUMAXL |
| LDUMAXB | Atomic unsigned maximum on byte | LDUMAXB, LDUMAXAB, LDUMAXALB, LDUMAXLB |
| LDUMAXH | Atomic unsigned maximum on halfword | LDUMAXH, LDUMAXAH, LDUMAXALH, LDUMAXLH |
| LDUMIN | Atomic unsigned minimum | LDUMIN, LDUMINA, LDUMINAL, LDUMINL |
| LDUMINB | Atomic unsigned minimum on byte | LDUMINB, LDUMINAB, LDUMINALB, LDUMINLB |
| LDUMINH | Atomic unsigned minimum on halfword | LDUMINH, LDUMINAH, LDUMINALH, LDUMINLH |
| STADD | Atomic add, without return | STADD, STADDL |
| STADDB | Atomic add on byte, without return | STADDB, STADDLB |
| STADDH | Atomic add on halfword, without return | STADDH, STADDLH |
| STCLR | Atomic bit clear, without return | STCLR, STCLRL |
| STCLRB | Atomic bit clear on byte, without return | STCLRB, STCLRLB |
| STCLRH | Atomic bit clear on halfword, without return | STCLRH, STCLRLH |
| STEOR | Atomic exclusive OR, without return | STEOR, STEORL |
| STEORB | Atomic exclusive OR on byte, without return | STEORB, STEORLB |
| STEORH | Atomic exclusive OR on halfword, without return | STEORH, STEORLH |
| STSET | Atomic bit set, without return | STSET, STSETL |
| STSETB | Atomic bit set on byte, without return | STSETB, STSETLB |
| STSETH | Atomic bit set on halfword, without return | STSETH, STSETLH |
| STMAX | Atomic signed maximum, without return | STSMAX, STSMAXL |
| STMAXB | Atomic signed maximum on byte, without return | STSMAXB, STSMAXLB |
| STMAXH | Atomic signed maximum on halfword, without return | STSMAXH, STSMAXLH |
| STMIN | Atomic signed minimum, without return | STSMIN, STSMINL |
| STMINB | Atomic signed minimum on byte, without return | STSMINB, STSMINLB |
| STMINH | Atomic signed minimum on halfword, without return | STSMINH, STSMINLH |
| STUMAX | Atomic unsigned maximum, without return | STUMAX, STUMAXL |
| STUMAXB | Atomic unsigned maximum on byte, without return | STUMAXB, STUMAXLB |
| STUMAXH | Atomic unsigned maximum on halfword,without return | STUMAXH, STUMAXLH |
| STUMIN | Atomic unsigned minimum, without return | STUMIN, STUMINL |
| STUMINB | Atomic unsigned minimum on byte, without return | STUMINB, STUMINLB |
| STUMINH | Atomic unsigned minimum on halfword,without return | STUMINH, STUMINLH |

### 2.2 Swap

交换指令仅支持一种寻址模式：Base register only。为了进行权限检查和观察点，所有交换
指令都被视为执行加载和存储。如果未实现FEAT_LSE2，则SWP指令需要自然对齐，而未对齐的
地址会生成对齐错误。

这些指令提供了排序选项，这些选项映射到体系结构中使用的acquire和release定义。具有
release语义的原子指令与Store-Release指令在多副本原子性方面具有相同的规则。

对于SWP指令，源寄存器和目标寄存器相同，如果指令生成同步数据中止，则源寄存器将恢复
到指令执行前保持的值。

| Mnemonic | Instruction   | See                        |
| -------- | ------------- | -------------------------- |
| SWP      | Swap          | SWP, SWPA, SWPAL, SWPL     |
| SWPB     | Swap byte     | SWPB, SWPAB, SWPALB, SWPLB |
| SWPH     | Swap halfword | SWPB, SWPAB, SWPALB, SWPLB |

### 2.3 Compare and Swap

比较与交换指令仅支持一种寻址模式：Base register only。为了进行权限检查和观察点，
所有比较与交换指令都被视为执行加载和存储。如果未实现FEAT_LSE2，则CAS指令要求自然
对齐，CASP指令要求与所访问的内存的总大小对齐。

这些指令提供了排序选项，这些选项映射到体系结构中使用的acquire和release定义。如果
比较和交换指令不执行存储，则无论指令排序选项如何，该指令都没有release语义。具有
release语义的原子指令与Store-Release指令在多副本原子性方面具有相同的规则。

对于CAS和CASP指令，体系结构允许数据读取清除与该位置有关联的任何独占监视器，即使
比较随后失败。如果这些指令生成同步数据中止，被比较和加载的寄存器恢复到指令执行之
前保存的值。

| Mnemonic | Instruction               | See                        |
| -------- | ------------------------- | -------------------------- |
| CAS      | Compare and swap          | CAS, CASA, CASAL, CASL     |
| CASB     | Compare and swap byte     | CASB, CASAB, CASALB, CASLB |
| CASH     | Compare and swap halfword | CASH, CASAH, CASALH, CASLH |
| CASP     | Compare and swap pair     | CASP, CASPA, CASPAL, CASPL |

## 3. 内核配置支持

### 3.1 内核配置

使能LSE需要开启以下内核配置:

- CONFIG_ARM64_LSE_ATOMICS
- CONFIG_ARM64_USE_LSE_ATOMICS (depends on CONFIG_JUMP_LABEL)

目前在openeuler_defconfig中，配置均默认打开:

```
CONFIG_ARM64_LSE_ATOMICS=y
CONFIG_ARM64_USE_LSE_ATOMICS=y
```

### 3.2 编译器支持

LSE使能需要编译器支持.

- GCC

当使用GCC编译器时, 需要支持以下选项:

```
-moutline-atomics
-mno-outline-atomics
```

以上配置帮助程序将在运行时确定是否可以使用ARMv8.1-A中的LSE指令；如果不能，他们将
使用基本ARMv8.0 ISA中存在的load/store-exclusive指令。

此选项仅适用于为基本ARMv8.0指令集编译时。如果使用较高的版本，例如-march=armv8.1-a
或-march=Armv8-a+lse，则将直接使用ARMv8.1-原子指令。当所选CPU支持“lse”功能时，使用-mcpu=lse也可以。默认情况下，此选项处于打开状态。

- Clang

当使用Clang编译器是, 需要支持-march=lse选项.

> [NOTE]
> commit e0d5896bd356 和 commit dd1f6308b28e修复了clang编译时使用lse的错误。

## 4. 软件接口

- 用户可以通过/proc/cpuinfo或HWCAP查询硬件对于LSE的支持情况.

## 5. 涉及代码和使能

| Commit id    | Subject                                                      | openeuler OLK5.10 enabled（Y/N） |
| ------------ | ------------------------------------------------------------ | -------------------------------- |
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
| c739dc83a0b6 | arm64: lse: rename ARM64_CPU_FEAT_LSE_ATOMICS for consistency | Y |
| 0e4a07092fc8 | arm64: kconfig: group the v8.1 features together | Y |
| 2e94da137903 | arm64: lse: use generic cpufeature detection for LSE atomics | Y |
| c1d7cd228b4b | arm64: spinlock: fix ll/sc unlock on big-endian systems | Y |
| 484c96dbb269 | arm64: lse: fix lse cmpxchg code indentation | Y |
| 7f08a414f29e | arm64: make ll/sc __cmpxchg_case_##name asm consistent | Y |
| 305d454aaa29 | arm64: atomics: implement native {relaxed, acquire, release} atomics | Y |
| da8d02d19ffd | arm64/capabilities: Make use of system wide safe value | Y |
| 57a65667991a | arm64: cmpxchg_dbl: fix return value type | Y |
| d86b8da04dfa | arm64: spinlock: serialise spin_unlock_wait against concurrent lockers | Y |
| cd5e10bdf379 | arm64: prefetch: don't provide spin_lock_prefetch with LSE | Y |
| ff96f7bc7bf6 | arm64: capabilities: Handle sign of the feature bit | Y |
| 5be8b70af1ca | arm64: lse: deal with clobbered IP registers after branch via PLT | Y |
| 3a5facd09da8 | arm64: spinlock: fix spin_unlock_wait for LSE atomics | Y |
| e490f9b1d3b4 | locking/atomic, arch/arm64: Implement atomic{,64}_fetch_{add,sub,and,andnot,or,xor}{,_relaxed,_acquire,_release}() | Y |
| 6822a84dd4e3 | locking/atomic, arch/arm64: Generate LSE non-return cases using common macros | Y |
| 2efe95fe6952 | locking/atomic, arch/arm64: Implement atomic{,64}_fetch_add,sub,and,andnot,or,xor}{,_relaxed,_acquire,_release}() for LSE instructions | Y |
| efd9e03facd0 | arm64: Use static keys for CPU features | Y |
| 05492f2fd87d | arm64: lse: convert lse alternatives NOP padding to use __nops | Y |
| 272d01bd790f | arm64: Fix circular include of asm/lse.h through linux/jump_label.h | Y |
| b7eed6ddaa71 | arm64: do not trace atomic operations | Y |
| 8997c93452d1 | arm64: atomic_lse: match asm register sizes | Y |
| 8df728e1ae61 | arm64: Remove redundant mov from LL/SC cmpxchg | Y |
| 32fb5d73c98b | arm64: atomics: Remove '&' from '+&' asm constraint in lse atomics | Y |
| 6b24442d68e7 | arm64: lse: Pass -fomit-frame-pointer to out-of-line ll/sc atomics | Y |
| 5b4747c5dce7 | arm64: capabilities: Add flags to handle the conflicts on late CPU | Y |
| 8a624f145c0d | arm64: lse: Include compiler_types.h and export.h for out-of-line LL/SC | Y |
| 32c3fa7cdf0c | arm64: lse: Add early clobbers to some input/output asm operands | Y |
| 7bd99b403405 | arm64: Kconfig: Enable LSE atomics by default | Y |
| 7c8fc35dfc32 | locking/atomics/arm64: Replace our atomic/lock bitop implementations with asm-generic | Y |
| 356c6fe7d80c | locking/atomics/arm64, arm64/bitops: Include <asm-generic/bitops/ext2-atomic-setbit.h | Y |
| 2a6c7c367de8 | arm64: lse: remove -fcall-used-x0 flag | Y |
| c0df10812835 | arm64, locking/atomics: Use instrumented atomics | Y |
| 5ef3fe4cecdf | arm64: Avoid redundant type conversions in xchg() and cmpxchg() | Y |
| b4f9209bfcd5 | arm64: Avoid masking "old" for LSE cmpxchg() implementation | Y |
| 959bf2fd03b5 | arm64: percpu: Rewrite per-cpu ops to allow use of LSE atomics | Y |
| 4230509978f2 | arm64: cmpxchg: Use "K" instead of "L" for ll/sc immediate constraint | Y |
| 6e4ede698d1c | arm64: percpu: Fix LSE implementation of value-returning pcpu atomics | Y |
| 34b8ab091f9e | bpf, arm64: use more scalable stadd over ldxr / stxr loop in xadd | Y |
| 16f18688af7e | locking/atomic, arm64: Use s64 for atomic64 | Y |
| 580fa1b87471 | arm64: Use correct ll/sc atomic constraints | Y |
| addfc38672c7 | arm64: atomics: avoid out-of-line ll/sc atomics | Y |
| 3337cb5aea59 | arm64: avoid using hard-coded registers for LSE atomics | Y |
| eb3aabbfbfc2 | arm64: atomics: Remove atomic_ll_sc compilation unit | Y |
| 0ca98b2456fb | arm64: lse: Remove unused 'alt_lse' assembly macro | Y |
| 0533f97b4356 | arm64: asm: Kill 'asm/atomic_arch.h' | Y |
| b32baf91f60f | arm64: lse: Make ARM64_LSE_ATOMICS depend on JUMP_LABEL | Y |
| 03adcbd996be | arm64: atomics: Use K constraint when toolchain appears to support it | Y |
| a48e61de758c | arm64: Mark functions using explicit register variables as '__always_inline' | Y |
| 395af861377d | arm64: Move the LSE gas support detection to Kconfig | Y |
| e0d5896bd356 | arm64: lse: fix LSE atomics with LLVM's integrated assembler | Y |
| dd1f6308b28e | arm64: lse: Fix LSE atomics with LLVM | Y |
