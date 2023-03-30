# WayCa-Kunpeng-故障处理-RAS特性介绍

## RAS基础

### 什么是RAS？

- Reliability—可靠性

指的是产品在规定的条件下和规定的时间内完成规定功能的能力。可靠性通常用FIT(Failure in Time)或MTBF（Mean Time Between Failure）来度量。

- Availability—可用性

指的是系统能够在给定的时间内确保可以运行的能力，即使系统出现一些小的问题也不会影响整个系统的正常运行，在某些情况下甚至可以进行 Hot Plug 的操作，替换有问题的组件，从而严格的确保系统的宕机时间在一定范围内。可用性通常用一定时间段内的宕机时间（如每年宕机n分钟/小时）或者系统实际运行时间的百分比（如5个9，99.999%）来衡量。

- Serviceability—可服务性

指的是系统能够提供便利的诊断功能，如系统日志，动态检测等手段方便管理人员进行系统诊断和维护操作，从而及早的发现错误并且修复错误，是支撑可维修（ **Maintainability** ） 达成的重要手段。

### 为什么需要RAS？

尽管硬件器件随着工艺，及技术的进步，可靠性技术也有较大的提升，但系统应用的软硬件规模越来越大，使得系统可靠性的挑战越来越大。

- 系统停机给用户业务造成的损失非常大

2016年ITIC调查发现，

98%的组织表示，停机一小时的成本超过10万美元；

81%的受访者表示，60分钟的停机时间花费了超过30万美元的业务费用

33%的企业报告说，一小时的停机时间使公司损失了100万美元超过500万美元。

- 系统故障发生的必然性

虽然故障很少，但企业系统可能非常庞大。所以发生故障是不可避免的。

- 系统可用性挑战

对于大规模应用的场景，比如HPC场景，单个节点故障，可能会导致整个业务重算，当集群规模达到一定程度，会导致整个系统趋于不可用。

## 功能描述

鲲鹏RAS从实现角度，分成4个大的组件：芯片、BIOS、OS和BMC。

- 芯片

芯片负责故障检测和纠正，并把错误记录下来，上报给软件，同时，对不可纠正错误进行抑制，防止错误影响其他模块，或者静默传播。

- BIOS

BIOS负责RAS功能的初始化使能，及RAS错误中断的统一处理，并通过APEI上报OS，和IPMB带外总线上报BMC两路进行上报，同时支持BMC的修复接口。

- OS

OS负责带内故障搜集，及错误修复动作，包括：1）支持固件优先故障处理方案，2）IO模块故障在线恢复，3）内存不可纠正错误memory failure处理，4）CPU执行路径故障处理，5）PCIe AER，6）内存可纠正错误在线Page隔离，7）rasdaemon支持的故障预测和隔离功能（内存和CPU核），8）故障信息带内收集。

- BMC

BMC负责带外故障搜集，及内存相关错误的在线预测和修复特性。

本文只描述OS支持的RAS特性。



## 特性详解

### 特性1：支持固件优先故障处理方案

- 特性详解：

该特性是固件优先方案的一部分，BIOS固件搜集完故障信息之后，会通过APEI的HEST表格，采用GHESv2格式，统一分级分类上报给OS。

根据错误对系统的影响可分为三级：CE、NFE和FE。

|      | 错误影响                                                     | OS处理                                                       |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| CE   | C硬件可纠正错误，对系统无影响。                              | 可以根据CE错误上报对故障进行预测，提前隔离可能发生更严重故障的硬件。 |
| NFE  | 硬件不可纠正错误，但只会影响部分功能，不会出现错误数据静默传播等致命问题。 | 支持IO模块在线恢复，内存memory failure处理。有些错误可能是偏软件的故障，可用于问题定位。 |
| FE   | 硬件不可恢复的错误，对系统影响非常大，可能会导致系统关键功能不可用，或者错误数据出现静默传播等致命问题。 | OS主动panic，防止给系统造成更严重损伤。                      |

根据硬件组件实现，对故障按模块进行分类，统一上报给OS，方便问题定位。其中Memory、PCIe和ARM Processor是标准模块，采用业界通用错误记录格式，其余采用鲲鹏自定义格式。

- 源码仓库：
https://gitee.com/openeuler/kernel/

- 特性代码目录：
drivers/acpi/apei/

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

- 回合的关键patches：<br/>
fe3f0ee8a5f5  - RAS: Report ARM processor information to userspace<br/>
924ceaed70a6  - ACPI / APEI: Notify all ras err to driver<br/>
afccb6485874  - ACPI: APEI: fix synchronous external aborts in user-mode

### 特性2：IO模块故障在线恢复

- 特性详解：

鲲鹏系列处理器集成的主要IO模块，都支持在线恢复功能，即当芯片检测到不可纠正错误时，可以通过驱动发起错误恢复，主要是进行模块安全复位，可以让模块在短时间内快速恢复。支持的模块包括：NIC、RoCE、SAS、ZIP、SEC和HPRE。

- 源码仓库：
https://gitee.com/openeuler/kernel/

- 特性代码：<br/>
NIC源码目录：drivers/net/ethernet/hisilicon/hns3/<br/>
RoCE源码目录：drivers/infiniband/hw/hns/<br/>
SAS源码目录：drivers/scsi/hisi_sas/<br/>
ZIP/SEC/HPRE源码目录：drivers/crypto/hisilicon/

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

- 回合的关键patch：<br/>
NIC patches：<br/>
842e0c3281e2 ("net: hns3: add error recovery module and type for himac")<br/>
579ffa72fbd1 ("net: hns3: add new ras error type for roce")<br/>
f5804f8184d2 ("net: hns3: Add configuration of TM QCN error event")<br/>
9e328d92bf28 ("net: hns3: add error handling compatibility during initialization")<br/>
9c28a3a6cf4e ("net: hns3: update error recovery module and type")<br/>
67abcd0c541f ("net: hns3: add support for imp-handle ras capability")<br/>
ce9918424850 ("net: hns3: add the RAS compatibility adaptation solution")<br/>
0a824254b900 ("net: hns3: add support for handling all errors through MSI-X")<br/>
b81168e14576 ("net: hns3: add scheduling logic for error handling task")<br/>
47b7b06483e5 ("net: hns3: add a separate error handling task")

RoCE patches：<br/>
3f324a33a5ee ("RDMA/hns: Recover 1bit-ECC error of RAM on chip")<br/>
d72651e67adf ("RDMA/hns: Remove unused abnormal interrupt of type RAS")

ZIP/SEC/HPRE patches：<br/>
1bd95691721a ("crypto: hisilicon/qm - disable queue when 'CQ' error")<br/>
a21f0b033677 ("crypto: hisilicon/qm - adjust order of device error configuration")<br/>
f27a89de78a5 ("crypto: hisilicon - enable new error types for QM")<br/>
209f72ee5caf ("crypto: hisilicon - add new error type for SEC")<br/>
d6ff658aa0c2 ("crypto: hisilicon - support new error types for ZIP")<br/>
0fc481e406d3 ("crypto: hisilicon/hpre - add two RAS correctable errors processing")<br/>
df1e4706d0a3 ("crypto: hisilicon/hpre - delete ECC 1bit error reported threshold")

### 特性3：内存不可纠正错误memory failure处理

- 特性详解：
Linux内核支持对内存不可纠正正错误的后处理，即memory_failure处理。该处理会尝试把出错内存对应的进程kill掉，并把该内存Page标记为poisoned，以避免该内存再被分配和使用。

- 源码仓库：
https://gitee.com/openeuler/kernel/

- 特性代码：
mm/memory-failure.c

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

### 特性4：CPU执行路径故障处理

- 特性详解：

当消费到不可纠正错误的数据之后，CPU核会触发SEA或者SEI，从而进入异常状态，并执行故障处理流程。

SEA上报的错误对应的指令地址是精确的，如果发生在用户态，可根据记录的物理地址进行应用级修复，或者直接kill 受影响的进程，如果发生在内核态，则由Linux内核主动执行panic操作，避免错误数据被使用。

SEI上报的错误对应的指令地址是不精确的，无法判断影响范围，不知道出错的数据是否已经扩散，只能由Linux内核主动执行panic，避免出现更严重问题。

- 源码仓库：
https://gitee.com/openeuler/kernel/

- 特性代码：
arch/arm64/mm/fault.c

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

### 特性5：PCIe AER

- 特性详解：
按照PCIe AER规范实现PCIe相关的检错特性，Linux内核支持其后处理，主要是通过link复位进行恢复。

- 源码仓库：
https://gitee.com/openeuler/kernel/

- 特性代码：
drivers/pci/pcie/

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

- 回合的关键patches：<br/>
4d25fd1e66e5 ("PCI/AER: Add RCEC AER error injection support")<br/>
b05f40fa3d26 ("PCI/AER: Add pcie_walk_rcec() to RCEC AER handling")<br/>
e149d280e516 ("PCI/ERR: Recover from RCiEP AER errors")<br/>
a34e781b667c ("PCI/ERR: Recover from RCEC AER errors")


### 特性6：内存可纠正错误在线Page隔离

- 特性详解：
按Page统计内存CE错误，当达到阈值之后，以page为单位对未使用的内存进行在线隔离，对正在使用的内存，等内存释放后再隔离。隔离支持配置阈值门限。

- 源码仓库：
https://gitee.com/src-openeuler/rasdaemon

- 特性代码：
ras-page-isolation.c

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

### 特性7：CPU核故障预测与在线隔离

- 特性详解：
对CPU核的健康状况进行检测，并隔离健康状况比较差的核。

- 源码仓库：
https://gitee.com/src-openeuler/rasdaemon

- 特性代码：
ras-cpu-isolation.c

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1

- 特别说明：
1、openEuler取的rasaemon主线v0.6.7的版本，尚未支持该特性，openEuler另外增加了patch来实现该功能的。
具体可参见rasdaemon.spec文件，里面有详细修改记录。<br/>
2、 依赖以下内核态patch<br/>
fe3f0ee8a5f5 ("RAS: Report ARM processor information to userspace")


### 特性8：故障信息带内收集

- 特性详解：

RAS错误信息收集，是问题定位和故障预测及处理的基础。鲲鹏系列处理器采用固件优先的RAS处理框架，通过APEI表格的GHESv2格式，把错误记录统一上报给Linux内核，Linux内核使用Trace point事件上报给用户态，最后由Linux社区通用RAS错误信息收集工具rasdaemon，完成收集。

- 源码仓库：
https://gitee.com/src-openeuler/rasdaemon

- 特性代码：
non-standard-hisilicon.c、non-standard-hisi_hip08.c

- 支持版本：
openEuler 22.03 lts、 openEuler 22.03 lts sp1
