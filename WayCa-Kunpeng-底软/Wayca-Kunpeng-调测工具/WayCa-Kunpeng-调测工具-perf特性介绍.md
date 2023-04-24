# openEuler WayCa SIG PERF 模块介绍

[TOC]

## 介绍

Perf 是 Linux 提供的一个性能分析工具，能够进行函数级与指令级的热点查找。Perf 是一个基于事件的工具，它可以利用 PMU，tracepoint 和内核中的特殊计数器来进行应用程序的事件统计，从而完成对处理器相关性能指标与操作系统相关性能指标的性能剖析。

Perf 工具的详细使用指导可以通过 help 获取。

当前 WayCa SIG PERF 支持平台为鲲鹏系列服务器，后续介绍的特性也都是基于鲲鹏系列服务器平台。

当前鲲鹏服务器上支持的 perf 硬件统计分为两类：

1> 第一类是 core PMU 事件，支持 arm 标准 PMU 事件，以及部分自定义的 PMU 事件，详细事件在用户手册中有提供。

2> 第二类是 uncore PMU 事件，依赖相应的 IO die PMU 设备和驱动。事件详细含义在用户手册中有提供。

## 特性

### Core PMU

ARM 提供了标准的 core PMU 事件定义和驱动，此外 core PMU 驱动也支持通过 event_id 来查询相应事件。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/arch/arm64/kernel/perf_event.c?id=f00fa5f4163b40c3ec8590d9a7bd845c19bf8d16)。

当前 openEuler 已经使能(f00fa5f4163b)。

linux config：

CONFIG_ARM_PMU

CONFIG_ARM_PMU_ACPI

###### 示例

查询事件

1> 标准事件: `perf list | grep -i armv8_pmuv3`。

2> 非标准事件: 用户手册。

### Uncore PMU

鲲鹏系列服务器上有多种 uncore PMU 硬件，每种 PMU 又会有多个设备（取决于 C die 和 IO die 数量），所以 uncore PMU 驱动会在 OS 中注册多个硬件设备，驱动对设备命名格式为 hisi\_<sccl/sicl>\<x>_\<device>\<y>，其中位于 CPU die 的设备为 sccl，位于 IO die 的设备为 sicl，device 为设备类型，x[2:0] 表示 die id，IO die A 的 id 是0，CPU die A 的 id 是1，IO die B 的 id 是2，CPU die B 的 id 是3。X[5:3]表示 Socket id 信息。

#### L3C PMU

###### 功能

L3 Cache 是片上一致性 Cache，主要包含维护一致性（directory）和缓存最近使用数据（cache）两大功能，该 cache 被 CPU 和各加速器、IO 共用，用以临时缓存在一段时间内会被多次使用的数据，这可以有效提高处理器及其他加速模块对外部存储器的访问带宽和访问效率。

L3C PMU 具有如下功能：

1> 统计本L3C的性能数据，包括命中率相关事件、带宽相关事件、平均延时事件、和CPU交互的事件、模块内部的事件；

2> 统计某个/某几个物理核或逻辑核对本L3C的访问操作、命中操作等性能数据；

3> 统计访问本L3C的读操作/写操作数；

4> 支持DataSource功能，统计返回CPU的数据来源，如来自本DIE的L3C，跨DIE的L3C，跨片的L3C，本DIE的DDR等的次数。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=2940bc4333707a05e69b3ffd737bda0dc0c3004f)。

当前 openEuler 已经使能(2940bc433370)。

Linux Config：

CONFIG_HISI_PMU

###### 示例

查询事件

`perf list | grep -i l3c`

统计事件

`perf stat -e hisi_sccl{X}_l3c{Y}/event_name/ -I 1000`

#### HHA PMU

HHA（Hydra Home Agent）是多个L3 cache之间或Device与L3 cache之间MESI一致性协议Hydra的Home Agent，可维护多个L3 cache之间或Device与L3 cache之间的数据一致性。最大支持16个Chip的一致性互联。HHA位于DDR与Ring Bus之间，为系统提供DDR访问通路，提供高带宽低延迟的DDR读写访问。

HHA PMU 具有如下功能：

1> 统计本HHA的性能数据，包括HHA接收到的事件（如HHA接收到来自跨片/本片跨DIE/CXL操作）、平均延时事件、和DDRC交互的事件、模块内部的事件；

2> 统计某个/某几个物理核或逻辑核对本HHA的访问指令、命中指令等性能数据；

3> 统计访问本HHA的读操作/写操作数；

4> 统计某个/某几个CCL/ICL访问HHA的次数。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=2bab3cf9104c5ab80a1b9c706d81d997548401e4)。

当前 openEuler 已经使能(2bab3cf9104c)。

Linux Config：

CONFIG_HISI_PMU

###### 示例

查询事件

`perf list | grep -i hha`

统计事件

`perf stat -e hisi_sccl{X}_hha{Y}/event_name/ -I 1000`

#### DDRC PMU

DDRC（DDR SDRAM Controller）实现对DDR的存取控制，该模块完成On-chip Network中传输的访问到DDR存储体的时序转换，并配合系统的QoS管理机制，对访问进行流量管理。DDRC PMU是一个平台设备，如果CPU DIE上使用的是DDR5，那么每个DIE上有6*2个DDRC PMU(双通道)，如果CPU DIE上使用的是DDR4，那么每个DIE上有4个DDRC PMU。

DDRC PMU 具有如下功能：

1> DDRC的带宽和延时；

2> DMC协议相关的事件和观测Rank/BG等切换的次数。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=904dcf03f086a2e3b9d1e02cb57c43ea2e588c8c)。

当前 openEuler 已经使能(904dcf03f086)。

Linux Config：

CONFIG_HISI_PMU

###### 示例

查询事件

`perf list | grep -i ddrc`

统计事件

`perf stat -e hisi_sccl{X}_ddrc{Y}/event_name/ -I 1000`

#### SLLC PMU

SLLC （Skyros Link Layer Controller）模块基于Skyros Link Layer协议，实现片内不同Die的系统总线之间的互联互通。

SLLC PMU 具有如下功能：

1> 统计跨die带宽、带宽利用率和时延的相关事件、模块内部的事件；

2> 统计某个/某几个物理核或逻辑核对本SLLC的访问数据；

3> 统计经过本SLLC的读操作/写操作数；

4> 统计某个/某几个CCL/ICL访问SLLC的次数；

5> 统计访问本SLLC的指令去往某个/某几个CCL/ICL的数量。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=3bf30882c3c7b6e376d9d6d04082c9aa2d2ac30a)。

当前 openEuler 已经使能(ed1d944f8400)。

Linux Config：

CONFIG_HISI_PMU

###### 示例

查询事件

`perf list | grep -i sllc`

统计事件

`perf stat -e hisi_sccl{X}_sllc{Y}/event_name/ -I 1000`

#### PA PMU

PA（Protocol Adapter）模块主要是完成片内和片间协议的适配功能。主要包括如下一些功能：片内和片间数据格式的转换，发送侧请求 buffer 的管理，DBID 管理，片内模块与 PA 之间请求 retry 的管理，和 Ring/HLLC 之间的流控处理，流量统计，低功耗管理等。

PA PMU 具有如下功能：

1> 统计跨片带宽、带宽利用率和时延的相关事件、跨片中断个数统计的事件；

2> 统计某个/某几个物理核或逻辑核对本PA的访问数据；

3> 统计经过本PA的读操作/写操作数；

4> 统计某个/某几个CCL/ICL访问PA的次数；

5> 统计访问本PA的指令去往某个/某几个CCL/ICL的数量。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=8404b0fbc7fbd42e5c5d28cdedd450e70829c77a)。

当前 openEuler 已经使能(ce0c00edc0da)。

Linux Config：

CONFIG_HISI_PMU

###### 示例

查询事件

`perf list | grep -i pa`

统计事件

`perf stat -e hisi_sccl{X}_pa{Y}/event_name/ -I 1000`

#### CPA PMU

CPA (CXL Protocol Agent)作为链路管理和协议处理模块，负责CXL协议和HCCS协议之间的适配，HCCS协议主要是完成片内一致性处理，CXL协议则主要完成芯片与片外加速器设备的互联。 CPA提供如下功能：报文格式转换，路由解析，ID重映射，flit的打包、组报，流量统计等。

CPA PMU 具有如下功能：

1> 支持统计CPA端口的带宽和延时相关的事件。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=6b79738b6ed91a2d0fe958819469eeedac3bca81)。

当前 openEuler 已经使能(d97253d6b9d9)。

Linux Config：

CONFIG_HISI_PMU

###### 示例

查询事件

`perf list | grep -i cpa`

统计事件

`perf stat -e hisi_sccl{X}_cpa{Y}/event_name/ -I 1000`

#### PCIe PMU

PCIe PMU支持统计标准PCIe设备的带宽、延时、带宽利用率等信息， 具有如下功能：

1> 统计每个PCIe CORE的带宽、延时、总线利用率等信息；

2> 统计某个/某几个root port的带宽、延时信息；

3> 统计某个EP设备的带宽、延时信息；

4> 统计大于/小于指定阈值长度的报文流量；

5> 满足触发条件后，统计的报文流量。

###### 使能

当前主线已经[使能](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/drivers/perf/hisilicon?id=8404b0fbc7fbd42e5c5d28cdedd450e70829c77a)。

当前 openEuler 已经使能(d7c5429e9e28)。

Linux Config：

CONFIG_HISI_PCIE_PMU

###### 示例

查询事件

`perf list | grep -i l3c`

统计事件

`perf stat -e hisi_pcie{X}_core{Y}/event_name/ -I 1000`
