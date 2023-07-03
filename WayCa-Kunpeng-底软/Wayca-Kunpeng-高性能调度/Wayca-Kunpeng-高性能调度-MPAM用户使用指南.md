# Wayca-Kunpeng-高性能调度-MPAM用户使用指南

当前Linux主线还未支持该特性

## 1. 介绍

MPAM（Memory Partitioning And Monitoring Extension）是Armv8.4体系结构引入的可选扩展。MPAM仅在AArch64下支持。MPAM扩展为内存系
统组件控件提供了一个框架，支持对组件的一个或多个资源进行分区。

MPAM软件包括两个部分：MPAM资源池的发现和初始化和Cache/memory资源的分区管理。MPAM当前通过Linux的resctrl（复用x86 RDT的对外接口）
进行控制。

Cache/memory带宽资源的分区管理部分承接用户态接口的输入输出和资源池之间的交互，该部分参考原x86 RDT的实现，并进行适当拓展，其中包
括：通过SMMU对IO增加partID标记接口，重新设计实现L2/L3 cdp机制，增加MPAM特有的L2/L3/MB优先级控制接口等。

当前Linux主线没有支持MPAM特性，openEuler实现了该特性，且当前仅为preview特性。关于MPAM在openEuler上支持及使能也可以参考
[openEuler社区仓库](https://gitee.com/openeuler/community)的MPAM文档sig/Kernel/mpam.md。

## 2. 内核相关配置

MPAM使能需要开启以下内核配置:

- CONFIG_MPAM
- CONFIG_ACPI_MPAM
- CONFIG_RESCTRL
- CONFIG_ACPI

## 3. 软件接口

- 单独添加mpam启动参数会使用默认方式启动mpam（很少用，只有特定鲲鹏服务器支持）
- 添加mpam=acpi启动参数会使用ACPI方式启动mpam

MPAM依赖resctrl实现对外接口，这是一个基于 kernfs 实现的操作接口，当系统没有默认挂载该文件系统时可以使用如下命令进行挂载：

```
mount –t resctrl resctrl /sys/fs/resctrl
```

默认情况下resctrl包含如下文件：

```
 cpus  cpus_list  info  mon_data  mon_groups  rmid  schemata  tasks
```

以上的挂载方式仅会使能默认的MPAM控制接口，其它的控制接口可以通过额外的挂载选项实现。
MPAM restrl可以接受如下挂载参数，以类似

```
mount -t resctrl resctrl /sys/fs/resctrl/ -o caPbm,caPrio,mbMax,mbMin,mbPrio
```

的形式进行挂载。

- Code Data Prioritization（CDP）：仅软件行为，业务使用Cache（包括L2，L3）时CODE段和DATA段分开控制。

- Cache Portion Bit Map（Cpbm）：按照位图控制分配特定容量和特定位置的Cache（包括L2和L3），其中每个bit代表一条cache way。

- Memory Bandwidth Maximum Partition（Max）：按照能够通过受控DMC组件最大带宽的百分比进行访存流量限制。

- Memory Bandwidth Minimum Partition（Min）：参照Max提供的最大带宽百分比，同时提供最小带宽百分比表示允许通过受控DMC组件的容量，
小于最小百分比将享受较高优先级的通过权。

- Hard Limit（Hdl）：开启会使得分区的带宽使用率降至最大带宽控制的范围之内，参考Max，否则，只有在通道拥挤时才会做适当限制。

- Priority Partition（Prio）：优先级不直接影响L3 Cache或带宽的资源分配，但是会在资源超分时影响资源分配，一个合适的优先级配比有
利于保证系统性能稳定。

例如使能l3的cdp功能:

```
mount -t resctrl resctrl /sys/fs/resctrl -o cdpl3
```

## 4. 涉及代码与使能

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

## 5.示例

使用默认参数挂载：

```
[root@localhost fs]#mount -t resctrl resctrl /sys/fs/resctrl
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3:0=fffffff;1=fffffff;2=fffffff;3=fffffff
MB:0=100;1=100;2=100;3=100
```

选择L3/MB控制方式:

```
[root@localhost fs]# mount -t resctrl resctrl /sys/fs/resctrl/ -o caPbm,caPrio,mbMax,mbMin,mbPrio
[root@localhost fs]# cat /sys/fs/resctrl/schemata
L3PBM:0=fffffff;1=fffffff;2=fffffff;3=fffffff
L3PRI:0=0;1=0;2=0;3=0
MBMAX:0=100;1=100;2=100;3=100
MBMIN:0=0;1=0;2=0;3=0
```
