# Wayca-Kunpeng-高性能调度-NMI用户使用指南

当前Linux主线已支持该特性

## 1. 介绍

在arm v8.8之前的版本中并不支持硬件的NMI中断，但是GIC支持中断的优先级功能，通过该特性可以模拟NMI中断。本文主要描述的是模拟NMI中断
（pseudo-NMI），当前主线及openEuler在鲲鹏上实现的也是pseudo-NMI中断。需要注意的是，不同于硬件NMI中断，pseudo-NMI实际上并不是完
全不可屏蔽的。
当前通过NMI中断在内核中主要实现两个功能：
- 通过NMI中断实现在中断上下文中的挂死检测。当挂死检测中断为非NMI中断时，系统如果在中断或者原子上下文中挂死则无法及时检测并复位。
- 通过NMI中断实现更加精准的PMU采样。当采样中断为非NMI中断时，无法采集中断或原子上下文中的信息。

## 2. 内核相关配置

NMI使能需要开启以下内核配置:

- CONFIG_ARM64_PSEUDO_NMI

## 3. 软件接口

pseudo-NMI中断默认不使能。需要通过启动参数进行使能：

```
irqchip.gicv3_pseudo_nmi=1
```

## 4. 涉及代码与使能

| COMMITID | SUBJECT | openeuler OLK-5.10 enabled（Y/N） |
| ---------- | ---------- | ----------- |
| f226650494c6a | arm64: Relax ICC_PMR_EL1 accesses when ICC_CTLR_EL1.PMHE is clear | Y |
| 8848f0665b3cd | arm64: Add cpuidle context save/restore helpers | Y |
| bc3c03ccb4641 | arm64: Enable the support of pseudo-NMIs | Y |
| b90d2b22afdc7 | arm64: cpufeature: Add cpufeature for IRQ priority masking | Y |
| 26dc129342cfc | irq: arm64: perform irqentry in entry code | Y |
| 133d05186325c | arm64: Make PMR part of task context | Y |
| 4d6a38da8e79e | arm64: entry: always set GIC_PRIO_PSR_I_SET during entry | Y |
| 09cf57eba3042 | KVM: arm64: Split hyp/switch.c to VHE/nVHE | Y |
| 336780590990e | irqchip/gic-v3: Support pseudo-NMIs when SCR_EL3.FIQ == 0 | Y |
