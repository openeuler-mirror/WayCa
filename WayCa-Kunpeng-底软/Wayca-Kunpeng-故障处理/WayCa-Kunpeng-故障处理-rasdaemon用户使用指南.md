# WayCa-Kunpeng-故障处理-rasdaemon用户使用指南

## 使用场景
Rasdaemon是社区通用的RAS故障管理工具。该工具运行在user space，通过trace event收集内核输出的RAS信息，收集到的错误信息，支持打印到终端和保存到sqlite3数据库。Rasdaemon也支持基于收集的个故障信息，进行故障预测，提前隔离潜在风险硬件组件。

鲲鹏系列芯片的CPU核故障（ARM Processor error）、内存故障（MC error）和PCIe故障（PCIe aer error）都会采用UEFI协议定义的标准错误格式上报，其余则都采用非标准错误格式上报，鲲鹏芯片自定义格式上报，包含以下4种格式：
- OEM format1：包含模块MN PLL SLLC AA SIOE POE DISP LPC GIC RDE SAS SATA USB；
- OEM format2：包含模块SMMU HHA PA HLLC DDRC L3TAG L3DATA；
- PCIe Local error：包含模块AP TL MAC DL SDI；
- Hisilicon Common Section：包含鲲鹏920系列处理器所有自定义模块。

Rasdaemon支持这4种格式错误的解析，使得错误记录更直观，容易理解。

## 硬件环境
基于鲲鹏920系列处理器的服务器及相关设备。

## 软件版本
openEuler 22.03 lts 、openEuler 22.03 lts sp1

## 安装使用

安装命令：
```
yum install rasdaemon
```

安装之后rasdaemon就自动启动了，也可以通过命令开启或者关掉
```
service radsaemon stop
service radsaemon start
```
通过sqlite3收集到日志，可以通过sqlite3命令全部dump出来，也可以使用ras-mc-ctl查询统计信息，例如：
```
sqlite3 /usr/local/var/lib/rasdaemon/ras-mc_event.db .dump
ras-mc-ctl --vendor-errors KunPeng9xx
```

如果需要手动构建，可以使用rpmbuild进行，具体请参考openEuler文档的“构建RPM包”相关章节。手动构建可以按自己需要，在rasdaemon.spec文件的configure环境使能或关闭一些功能。其中一些关键的编译选项如下：
- --enable-mce： 使能内存错误收集
- --enable-aer： 使能PCIe AER错误收集
- --enable-sqlite3： 使能sqlite3数据记录
- --enable-diskerror：使能硬盘故障错误记录
- --enable-non-standard：使能非标准错误收集
- --enable-hisi-ns-decode：使能鲲鹏芯片自定义错误格式的错误记录解析
- --enable-arm： ARM Processor错误记录收集
- --enable-memory-failure：使能内存memory failure处理结果状态信息收集
- --enable-memory-ce-pfa： 使能基于内存CE错误的故障预测及在线隔离功能
- --enable-cpu-fault-isolation： 使能CPU核故障在线预测和隔离功能

## 依赖
- 内核Config选项
Linux内核需要打开以下配置。内核配置文件路径：kernel/arch/arm64/configs/。

```
CONFIG_ACPI_APEI=y
CONFIG_ACPI_APEI_GHES=y
CONFIG_ACPI_APEI_EINJ=y
CONFIG_ACPI_APEI_MEMORY_FAILURE=y
CONFIG_ACPI_APEI_PCIEAER=y
CONFIG_ACPI_APEI_SEA=y
CONFIG_ACPI_HED=y
CONFIG_ACPI_REDUCED_HARDWARE_ONLY=y
CONFIG_EDAC_GHES=y
CONFIG_TRACEPOINTS=y
```
