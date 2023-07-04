# Wayca-Kunpeng-高性能调度-TWED用户使用指南

当前Linux主线已支持该特性

## 1. 介绍

armv8.7引入TWED特性来延迟WFE指令的捕获，WFE指令在所有异常级别都可用。通过在EL0、EL1或EL2执行的软件尝试进入低功耗状态，可以配置
为陷阱到更高的异常级别。如果FEAT_TWED使能，那么可以配置WFE陷阱前的延迟。如果配置了WFE陷阱前的延迟，则延迟不会影响陷阱的优先级。

## 2. 内核相关配置

使能TWED特性需要打开配置CONFIG_ARM64_TWED，OLK5.10中默认为y。

## 3. 编译器支持

TWED使能需要编译器支持

- GCC

当使用GCC编译器时, 需要支持以下选项
-march=armv8.7-a或-march=Armv8-a+twed

- Clang

当使用Clang编译器时, 需要支持-march=twed 选项

## 4. 软件接口

用户可以通过/proc/cpuinfo或HWCAP或者查询ID_AA64MMFR1_EL1.TWED字段查询硬件对于TWED的支持情况。

## 5. 涉及代码和使能

| COMMITID | SUBJECT | openeuler OLK-5.10 enabled（Y/N） |
| ---------- | ---------- | ----------- |
| 9c8b91e8dbf72 | KVM: arm64: Make use of TWED feature | Y |
| 1d9393307f4f4 | arm64: cpufeature: TWED support detection | Y |
