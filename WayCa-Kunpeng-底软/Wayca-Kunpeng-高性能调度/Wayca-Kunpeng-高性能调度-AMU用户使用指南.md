# Wayca-Kunpeng-高性能调度-AMU用户使用指南

当前Linux主线已支持该特性

## 1. 介绍

处理器包括基于AMUv1体系结构的活动监控。它旨在用于系统管理，而性能监控则针对用户和调试应用程序。活动监视器为系统电源管理和持续监控提供了有用的信息。活动监视器在操作中是只读的,它们的配置仅限于实现的最高异常级别。

## 2. 内核相关配置

AMU使能需要开启以下内核配置:
- CONFIG_ARM64_AMU_EXTN

## 3. 软件接口

与PMU类似, AMU提供多个硬件计数器用于采集硬件事件, 按照实现分为两组计数器:
- 架构计数器组:
  包含4个用于统计固定事件的计数器:
  | Event | Event Code |
  |-------|------------|
  |CPU_CYCLES|0x0011|
  |CNT_CYCLES|0x4004|
  |INST_RETIRED|0x0008|
  |STALL_BACKEND_MEM|0x4005|
- 辅助计数器组:
  Armv8允许实现最多16个计数器, 每个计数器可编程或统计固定事件. 当前系统可支持4个可编程的辅助计数器.

硬件通过AMCFGR_EL0描述AMU的支持情况, 包括支持的计数器组数量及其它特性等.
硬件通过AMCGCR_EL0描述每组实现的计数器数量.
可以通过AMCNTEN{SET, CLR}{0, 1}_EL0对计数器进行禁用/使能.
AMU支持用户态访问. 通过AMUSERENR_EL0控制是否捕获EL0下的AMU访问操作. 在当前的主线内核中, 处于安全考虑访问权限未开放给用户态. 因此当前不支持在用户态访问AMU硬件.

## 4. 涉及代码与使能

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
