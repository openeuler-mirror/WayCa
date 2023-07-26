# openEuler WayCa SIG 鲲鹏基础IO SAS模块介绍
# 一、SAS驱动简介
SAS驱动是适配920系列处理器SAS控制器的驱动模块。SAS（Serial Attached SCSI）即串行SCSI技术，一种磁盘连接技术。SAS控制器用于磁盘与内存之间进行交互。

## 源码获取路径
- Linux kernel仓库：https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/
- openEuler 仓库：https://gitee.com/openeuler/kernel.git
- 源码路径：driver/scsi/hisi_sas

## 功能描述
对于鲲鹏920系列SAS模块，支持的特性如下：
- 兼容SAS3.0协议，同时向下兼容SAS2.0和SAS1.0
-支持SSP/SMP/STP
-支持连接SATA盘（非AHCI方式）
-支持SATA协议定义的NCQ
-支持最多1024个设备
-支持最多并发4096个IO
-支持宽端口（最多每个端口8个PHY）和窄端口模式
-支持SAS链接速率12/6/3/1.5Gbit/s自协商， SATA链接速率6/3/1.5Gbit/s自协商
-支持热插拔
-支持SAS DIF
-支持SAS MSI中断模式及中断聚合
-支持逻辑侧单个IO或整个盘的IO的abort
-支持错误以RAS方式上报
-支持DFX
-支持BIST
-支持直连LED
-支持SAS runtime PM

## 规格描述

本节主要描述用户可见、可感知的规格，以及规格描述、主线支持版本的列表

| 规格名称 | 规格描述 | 主线支持版本 | 备注 |
| ------------ | ------------ | ------------ | ------------ |
| 兼容SATA盘 | 非AHCI方式，可以直连SATA盘或expander连接SATA盘 | Linux Kernel 4.13 | 基本功能 |
| 支持SSP/SMP/STP | 支持直连或expander<br>连接SAS盘和SATA盘 | Linux Kernel 4.13 | 基本功能 |
| 支持最多1024个设备 | 支持最大设备数为1024个 | Linux Kernel 4.13 | 基本功能 |
| 支持最多4096个IO | 支持最多4096个IO | Linux Kernel 4.13 | 基本功能 |
| 支持DIF | 支持SAS DIF特性 | Linux Kernel 4.21 | - |
| 支持DIX | 支持SAS DIX特性 | Linux Kernel 5.1 | 依赖于DIF |
| 中断聚合 | 支持SAS MSI中断聚合 | Linux Kernel 4.21 | - |
| 支持错误以RAS方式上报 | 支持错误以RAS方式上报 | Linux Kernel 5.2 | - |
| 支持DFX | 以debugfs方式实现 | Linux Kernel 4.22 （Basic）<br>Linux Kernel 5.5 (Enhance) | - |
| 支持BIST | 码流环回功能 | Linux Kernel 5.4 | - |
| 支持LED | 直连LED点灯 | Linux Kernel 4.18 | - |
| 支持SAS controller runtime PM | 支持SAS控制器的SAS controller runtime PM | Linux Kernel 4.16 | - |
## 内核config选项
SAS驱动涉及的内核配置选项：CONFIG_SCSI_HISI_SAS和CONFIG_SCSI_HISI_SAS_PCI。对于鲲鹏920系列处理器使能，需要同时配置CONFIG_SCSI_HISI_SAS和CONFIG_SCSI_HISI_SAS_PCI。
## 加载依赖
模块编译后，会生成两个KO文件：hisi_sas_main.ko、hisi_sas_v3_hw.ko。先加载hisi_sas_main.ko，再加载hisi_sas_v3_hw.ko。
## 模块参数
SAS驱动模块当前包含四个模块参数，针对hisi_sas_main.ko的模块参数是：debugfs_enable、debugfs_dump_count，针对hisi_sas_v3_hw.ko的模块参数是：prot_mask、intr_conv。

| 模块参数  | 含义  | 默认值  |  设置 |
| ------------ | ------------ | ------------ | ------------ |
|  debugfs_enable |  是否使能debugfs | 1 | debugfs_enable=1使能debugfs功能<br>debugfs_enable=0不使能debugfs功能  |
|debugfs_dump_count|debugfs dump次数设置|1|debugfs_dump_count = n<br>(0<=n<=50)|
|  prot_mask | DIF/DIX类型设置  | 0  | n: 设置DIF/DIX类型；<br>0: 不使能DIF/DIX  |
| intr_conv  | 是否使能中断聚合  | 0  | intr_conv =1使能中断聚合<br>intr_conv =0不使能中断聚合  |


# 二、特性详解

## 特性1：中断聚合
- 特性介绍

上层的每个IO下发在底层驱动是通过DQ队列（即发送队列）发送给SAS控制器，IO完成后通过CQ队列返回。CQ队列的返回以中断方式触发。当IO产生比较频繁时，对应中断产生会很频繁，此时通过将部分中断一起上报可以提升性能。SAS驱动支持两种类型的中断聚合：

1.中断聚合类型1：

所有CQ队列（即CQ0-CQ15）的中断全部在CQ0上产生。

2.中断聚合类型2：

在规定时间内对本队列产生的中断进行聚合，如果在规定时间内聚合到指定数量则产生一次中断，如果在规定时间内没有聚合到指定数量的中断，在规定时间达到时产生中断。

- 涉及代码与使能

| commitid  | subject  |openeuler OLK-5.10 enabled(Y/N) |
| ------------ | ------------ | ------------ |
| 37359798ec44  |scsi: hisi_sas: Add support for interrupt coalescing for v3 hw  | Y  |
| c3566f9a617d  |scsi: hisi_sas: Create separate host attributes per HBA  | Y  |
| 488cf558e3d7  |scsi: hisi_sas: Add support for interrupt converge for v3 hw  | Y  |

- 示例

使能中断聚合类型1：

默认不使能中断聚合类型1，用户可以通过模块参数使能。
```cpp
insmod hisi_sas_v3_hw.ko intr_conv=1
```
使能中断聚合类型2（配置中断聚合时间和次数）：
```cpp
echo 8 > /sys/devices/pci0000:74/0000:74:02.0/host0/scsi_host/host0/intr_coal_count
//设置中断聚合次数为8次
echo 1000 > /sys/devices/pci0000:74/0000:74:02.0/host0/scsi_host/host0/intr_coal_ticks
//设置中断聚合时间为1ms
```
查看中断聚合状态：

对于中断聚合类型1，可通过PCI设备SYSFS接口目录下intr_conv_v3_hw接口查询中断聚合类型1是否使能；
对于中断聚合类型2，输入的中断聚合时间和中断聚合次数，其中任意参数为0，则表示不使能中断聚合类型2。




## 特性2：DIF/DIX功能
- 特性介绍

为了保证从硬盘到控制器再到应用程序之间的数据完整性，Linux内核和存储厂商共同推出了DIF+DIX来保证从应用程序到磁盘之间的完整IO路径上的“端对端”数据保护。

DIF(Data Integrity Field)允许提供在主机适配器和磁盘之间交换额外的保护信息。DIX(Data Integrity Extension)提供应用程序到主机适配器之间的保护信息。在传输过程中在数据IO后追加一些额外的信息，来检查数据的完整性，这些额外信息即为完整性保护信息。对于磁盘，会在原来扇区大小（512字节或者4K字节）后增加8字节来存储扇区数据的保护信息。

用户可通过模块参数prot_mask决定DIF/DIX支持模式。通过sg_utils工具，可以查询SAS盘的DIF支持情况、格式化为支持的DIF格式或恢复为普通盘。


- 涉及代码与使能

| commitid  | subject  |openeuler OLK-5.10 enabled(Y/N) |
| ------------ | ------------ | ------------ |
| 6db831f4ef76  | scsi: hisi_sas: Make sg_tablesize consistent value  | Y  |
| 6e1b731b5352  | scsi: hisi_sas: Relocate some code to reduce complexity  | Y  |
| d6a9000b81be  | scsi: hisi_sas: Add support for DIF feature for v2 hw  | Y |
| b3cce125cb1e  | scsi: hisi_sas: Add support for DIX feature for v3 hw  | Y  |

- 示例

配置DIF/DIX支持模式

```c
insmod hisi_sas_v3_hw.ko prot_mask=0x70
//支持DIX0/DIX1/DIX2/DIX3
insmod hisi_sas_v3_hw.ko prot_mask=0x77
//支持DIF1/DIF2/DIF3，支持DIX0/DIX1/DIX2/DIX3
```

查询SAS盘对DIF的支持
```c
sg_vpd --page=ei --long /dev/sdx |grep SPT
```
格式化DIF SAS盘
```c
sg_format --format --fmtpinfo=2 /dev/sdx  //对于DIF1格式的盘
sg_format --format --fmtpinfo=3 --pfu=1 /dev/sdx  //对于DIF3格式的盘
```
恢复DIF盘为正常盘
```c
sg_format -F -s 512 /dev/sdx
```

- 功能限制

EXT2文件系统由于其下发I/O数据存在并发性，导致数据与CRC之间不匹配，因此EXT2文件系统格式不支持DIF功能。


## 特性3：debug调测功能
- 特性介绍

debug调测功能为用户提供异常场景下自动或手动dump寄存器及部分IO数据的能力。SAS控制器驱动通过debugfs目录提供debug调测功能接口。其debugfs目录路径是：/sys/kernel/debug/hisi_sas，在该目录下会生成以SAS控制器BDF号命名的子目录。

- dump功能

自动dump功能在SAS控制器出现严重错误时（如internal abort命令超时）触发。
手动dump可以在debugfs目录中通过命令 echo 1 > trigger_dump 对debugfs信息进行手动dump，完成dump后会在/sys/kernel/debug/hisi_sas/BDF_number/dump目录下生成相应信息。

- debugfs dump接口展示

![debugfs dump接口展示](http://image.huawei.com/tiny-lts/v1/images/adb310517372e6815b81e124704c972c_622x619.png)
- FIFO功能

FIFO功能是在接收硬件发出的触发信号后，收集相应的链路信息，并通过debugfs接口呈现给用户。FIFO功能为每个PHY提供7个配置接口以及1个展示dump数据的接口。

|	接口名称	|	接口描述	|	接口配置	|
| ------------ | ------------ | ------------ |
|	signal_sel	|	通过signal_sel可以选择一个指定的信号组	|	 b0000: 速率协商打包信号组<br>b0001: 12G tx train本端训练对端的调整方式，以及与HiLink的交互信号<br>b0010: 12G tx train对端训练本端的调整方式<br>b0011: sas_phy_ctrl_dfx0<br>b0100: sas_phy_ctrl_dfx1<br>b0101: sas_pcs_dfx0<br>other: 速率协商打包信号组	|
|	dump_msk	|	选择debug信号组需要dump的数据	|	 -	|
|	dump_mode	|	配置信号触发后数据写入模式	|	b001: 一直dump（无需配置trigger/ trigger_msk/trigger_mode接口）<br>b010: 触发条件触发后，再dump32个数据就停止<br>b100: 触发条件触发后，不再dump 	|
|	trigger_mode	|	配置触发模式	|	b001: 当cfg_dfx_trigger_msk对应的bit为1的debug信号发生跳变时触发（无需配置trigger接口）<br>b010: 当debug信号等于cfg_dfx_trigger（cfg_dfx_trigger_msk对应的bit为1）时触发<br>b100: 当debug信号不等于cfg_dfx_trigger（cfg_dfx_trigger_msk对应的bit为1）时触发trigger	|
|	trigger	|	通过trigger指定当前的信号组中触发信号的有效触发电平	|	 -	|
|	trigger_msk	|	通过trigger_msk选择屏蔽一部分触发信号	|	 -	|
|	update_config	|	更新接口配置	|	b001:更新FIFO配置参数	|
|	fifo_data	|	展示根据用户配置dump的数据	|	NA 	|


- 涉及代码与使能

| commitid  | subject  |openeuler OLK-5.10 enabled(Y/N) |
| ------------ | ------------ | ------------ |
| cd96fe600cc4 | scsi: hisi_sas: Add trace FIFO debugfs support | Y |
| 35160421b63d | scsi: hisi_sas: Don't create debugfs dump folder twice | Y |
| d28ed83b7693 | scsi: hisi_sas: Add timestamp for a debugfs dump | Y |
| 35ea630b2bad | scsi: hisi_sas: Add debugfs file structure for CQ | Y |
| 1b54c4db725d | scsi: hisi_sas: Add debugfs file structure for DQ | Y |
| c61163981076 | scsi: hisi_sas: Add debugfs file structure for registers | Y |
| 1f66e1fd26bd | scsi: hisi_sas: Add debugfs file structure for port | Y |
| e15f2e2dff5b | scsi: hisi_sas: Add debugfs file structure for IOST | Y |
| 0161d55f23a1 | scsi: hisi_sas: Add debugfs file structure for ITCT | Y |
| b714dd8f36dc | scsi: hisi_sas: Add debugfs file structure for IOST cache | Y |
| 357e4fc7a933 | scsi: hisi_sas: Add debugfs file structure for ITCT cache | Y |
| a70e33eae363 | scsi: hisi_sas: Allocate memory for multiple dumps of debugfs | Y |
| 905ab01faf5f | scsi: hisi_sas: Add module parameter for debugfs dump count | Y |
| 8f6432986e61 | scsi: hisi_sas: Add ability to have multiple debugfs dumps | Y |
| cabe7c10c97a | scsi: hisi_sas: Delete the debugfs folder of hisi_sas when the probe fails | Y |
| f873b66119f2 | scsi: hisi_sas: Record the phy down event in debugfs | Y |
| 26f84f9bc3ba | scsi: hisi_sas: Code style cleanup | Y |
| b601577df68a | scsi: hisi_sas: Add missing newlines | Y |
| ca06f2cd01d0 | scsi: hisi_sas: Make phy index variable name consistent | Y |
| caeddc0453b9 | scsi: hisi_sas: Do not modify upper fields of PROG_PHY_LINK_RATE reg | Y |
| 4b3a1f1feda6 | scsi: hisi_sas: Modify macro name for OOB phy linkrate | Y |
| 623a4b6d5c2a | scsi: hisi_sas: Move debugfs code to v3 hw driver | Y |
| 1dbe61bf7d76 | scsi: hisi_sas: Enable debugfs support by default | Y |
| ef63464bcf8f | scsi: hisi_sas: Create root and device debugfs directories | Y |
| eb1c2b72b769 | scsi: hisi_sas: Alloc debugfs snapshot buffer memory for all registers | Y |
| 49159a5e4175 | scsi: hisi_sas: Take debugfs snapshot for all regs | Y |
| caefac199676 | scsi: hisi_sas: Debugfs global register create file and add file operations | Y |
| 971afae7cf4f | scsi: hisi_sas: Add debugfs CQ file and add file operations | Y |
| 148e379f60c5 | scsi: hisi_sas: Add debugfs DQ file and add file operations | Y |
| 1afb4b852479 | scsi: hisi_sas: Add debugfs IOST file and add file operations | Y |
| 5979f33b982d | scsi: hisi_sas: Add debugfs ITCT file and add file operations | Y |
| 7c5e13636391 | scsi: hisi_sas: Add manual trigger for debugfs dump | Y |
| 61a6ebf3f584 | scsi: hisi_sas: Add debugfs for port registers | Y |
| bbe0a7b348b3 | scsi: hisi_sas: Snapshot HW cache of IOST and ITCT at debugfs | Y |
| b0b3e4290e28 | scsi: hisi_sas: Snapshot AXI and RAS register at debugfs | Y |
| 7105e68afaec | scsi: hisi_sas: add debugfs auto-trigger for internal abort time out | Y |
| 7ec7082c57ec | scsi: hisi_sas: Add hisi_sas_debugfs_alloc() to centralise allocation | Y |

- 示例

```c
echo 0 > /sys/kernel/debug/hisi_sas/0000:30:04.0/fifo/0/signal_sel
//指定为速率协商打包信号组
echo 0x7ff > /sys/kernel/debug/hisi_sas/0000:30:04.0/fifo/0/dump_msk
//指定需要监测的BIT
echo 1 > /sys/kernel/debug/hisi_sas/0000:30:04.0/fifo/0/dump_mode
//指定数据写入模式为一直dump
echo 1 > /sys/kernel/debug/hisi_sas/0000:30:04.0/fifo/0/update_config
//更新接口配置
echo 12.0 Gbit > /sys/class/sas_phy/phy-4:0/maximum_linkrate
//触发速率重新协商
cat /sys/kernel/debug/hisi_sas/0000:30:04.0/fifo/0/fifo_data
//展示dump信息
```

## 特性4：BIST功能
- 特性介绍

鲲鹏920系列处理器SAS支持环回码流功能，此功能通过设置链接速率、码型、PHY ID、环回类型等参数让控制器产生对应的码流，从而通过产生的码流检验链路的质量以及控制器链路层和PHY层的逻辑速率的正确性。

- BIST功能接口展示

BIST功能依赖于debugfs，因此要使能BIST功能需要提前使能debugfs。在BIST使能后会生成如下文件。
![图片说明](http://image.huawei.com/tiny-lts/v1/images/04fb39cd3444fde201f75bf7e35cb19a_767x814.png)

对于SAS BIST，组网要求与正常SAS控制器使用不一样：除remote模式外，其它模式不连接任何设备；remote模式需外接SAS环回头。

- 涉及代码与使能

| commitid  | subject  |openeuler OLK-5.10 enabled(Y/N) |
| ------------ | ------------ | ------------ |
|97b151e75861|scsi: hisi_sas: Add BIST support for phy loopback|Y|
|65a3b8bd5694|scsi: hisi_sas: Set the BIST init value before enabling BIST|Y|
| 981cc23e741a | scsi: hisi_sas: Add BIST support for fixed code pattern | Y |
| 2c4d582322ff | scsi: hisi_sas: Add BIST support for phy FFE | Y |

- 示例
```c
echo 1 > /sys/kernel/debug/hisi_sas/0000:30:04.0/bist/phy_id
//设置需要检测的PHY为phy1
echo 3.0 Gbit > /sys/kernel/debug/hisi_sas/0000:30:04.0/bist/link_rate
//重新设置phy1的链路速率
echo 1 > /sys/kernel/debug/hisi_sas/0000:30:04.0/bist/enable
//使能BIST
echo 0 > /sys/kernel/debug/hisi_sas/0000:30:04.0/bist/enable
//关闭BIST
cat /sys/kernel/debug/hisi_sas/0000:30:04.0/bist/cnt
//查看phy1的误码数量
```

- 测试限制

		对于serdes方式，当前serdes工作模式并不是环回，会一直产生误码；
		对于码型为FIXED_DATA时, 不会有误码产生，不需要关注；
		可以通过误码注入，检查是否产生误码；
		由于协商速率要求，测试第一次需要将速率设置为大于等于3.0Gbit，后续可自由设置。


## 特性5：runtime PM
- 特性介绍

SAS驱动支持SCSI设备的runtime PM。用户可以通过sysfs接口使能SCSI设备的runtime PM，并设置设备处于空闲状态达到用户指定时间N秒后，自动进入runtime PM状态，当用户给处于runtime PM状态的SCSI设备发送业务时，能够自动唤醒SCSI设备；当SCSI设备再次空闲N秒后自动进入runtime PM状态。

驱动支持SAS 控制器的runtime PM。用户可以通过sysfs接口使能SAS 控制器的runtime PM，当SAS 控制器连接的所有SCSI设备都进入runtime PM状态后，SAS 控制器会进入runtime PM状态；只要有一个SCSI设备不是runtime PM状态，SAS 控制器不会进入runtime PM状态。SAS 控制器进入runtime PM状态后，用户可对已进入runtime PM状态的SCSI设备下发业务，唤醒SCSI设备和SAS 控制器，当没有业务时SAS控制器和SCSI设备又会自动进入runtime PM状态。

- 用户接口及说明

| Runtime PM SYSFS用户接口  | 接口类型  | 含义及取值  | 默认值  |
| ------------ | ------------ | ------------ | ------------ |
| control  | rw  |  设置是否自动runtime PM<br>auto: 允许设备runtime PM<br>on: 不允许设备runtime PM | scsi设备:on<br>scsi控制器:auto  |
| runtime_status  | r  | 表示设备当前runtime状态<br>suspended: 进入suspended状态<br>active: 唤醒状态<br>suspending：正在suspending过程中<br>resuming: 正在唤醒过程中<br>unsupported: 不支持runtime PM  | active  |
| runtime_suspended_time  | r  | 设备处于suspended状态的时间  | NA  |
| runtime_active_time  | r  | 设备处于active状态的时间  |  NA |
| autosuspend_delay_ms  | rw  |若用户使能自动runtime PM，此项设置进入runtime PM状态前设备空闲的时间 <br>-1: 一直不休眠<br>n: 设备空闲n秒后自动进入休眠| -1 |

- 涉及代码与使能

驱动支持SCSI设备的runtime PM

| commitid  | subject  |openeuler OLK-5.10 enabled(Y/N) |
| ------------ | ------------ | ------------ |
| 6c459ea1542b | scsi: hisi_sas: Switch to new framework to support suspend and resume | Y |
| 65ff4aef7e9b | scsi: hisi_sas: Add controller runtime PM support for v3 hw | Y |
| e06596d5000c | scsi: hisi_sas: Add check for methods _PS0 and _PR0 | Y |
| 16fd4a7c5917 | scsi: hisi_sas: Add device link between SCSI devices and hisi_hba | Y |
| b14a37e011d8 | scsi: hisi_sas: Filter out new PHY up events during suspend | Y |
| 69f4ec1edb13 | scsi: hisi_sas: Recover PHY state according to the status before reset | Y |
| 027e508aea45 | scsi: hisi_sas_v3_hw: Don't use PCI helper functions | Y |
| 71c8f15e1dbc | scsi: hisi_sas_v3_hw: Remove extra function calls for runtime pm | Y |
| 17b5e4d14837 | scsi: hisi_sas_v3_hw: Drop PCI Wakeup calls from .resume | Y |
| 027e508aea45 | scsi: hisi_sas_v3_hw: Don't use PCI helper functions | Y |
| 71c8f15e1dbcd | scsi: hisi_sas_v3_hw: Remove extra function calls for runtime pm | Y |
| 17b5e4d14837 | scsi: hisi_sas_v3_hw: Drop PCI Wakeup calls from .resume | Y |

驱动支持SAS 控制器的runtime PM

| commitid  | subject  |openeuler OLK-5.10 enabled(Y/N) |
| ------------ | ------------ | ------------ |
| d6e366685981  | PM: runtime: Drop pm_runtime_clean_up_links()  | Y  |
| e0e398e20463 |PM: runtime: Drop runtime PM references to supplier on link removal| Y  |
| 9226c504e364 |PM: runtime: Resume the device earlier in __device_release_driver()| Y  |
| fbefe22811c3  |scsi: libsas: Don't always drain event workqueue for HA resume   |  Y |
| 6cc739087784  |scsi: Revert "scsi: hisi_sas: Filter out new PHY up events during suspend"   |  Y |
| 97f410093984  |scsi: hisi_sas: Add more logs for runtime suspend/resume   | Y  |
| 29e2bac87421  |scsi: hisi_sas: Fix some issues related to asd_sas_port->phy_list | Y  |
| 6e1fcab00a23  |scsi: block: pm: Always set request queue runtime active in blk_post_runtime_resume()  |  Y |
| 42159d3c8d87  |scsi: libsas: Add spin_lock/unlock() to protect asd_sas_port->phy_list   | Y  |
| 133b688b2d03  |scsi: mvsas: Add spin_lock/unlock() to protect asd_sas_port->phy_list   | Y  |
| e31e18128eb9  |scsi: libsas: Insert PORTE_BROADCAST_RCVD event for resuming host | Y  |
| 0da7ca4c4fd9  |scsi: libsas: Resume host while sending SMP I/Os   |  Y |
| 4ea775abbb5c  |scsi: libsas: Add flag SAS_HA_RESUMING   | Y  |
| 1bc35475c6bf  |scsi: libsas: Refactor sas_queue_deferred_work()   | Y  |
| bf19aea4607c  |scsi: libsas: Defer works of new phys during suspend   | Y  |
| 307d9f49cce9  |scsi: libsas: Keep host active while processing events   | Y  |
| 36c6b7613ef1  |scsi: hisi_sas: Initialise devices in .slave_alloc callback   | Y  |
| 046ab7d0f594  |scsi: hisi_sas: Wait for phyup in hisi_sas_control_phy()|Y|

- 示例

```c
SCSI设备设置进入runtime PM状态
echo auto > /sys/devices/pci0000:b4/0000:b4:02.0/host6/port-6:0/end_device-6:0/target6:0:0/6:0:0:0/power/control
//使能SCSI设备自动runtime PM
echo 5000 > /sys/devices/pci0000:b4/0000:b4:02.0/host6/port-6:0/end_device-6:0/target6:0:0/6:0:0:0/power/autosuspend_delay_ms
//设置SCSI设备进入runtime PM状态前设备空闲的时间为5s

SAS控制器设置进入runtime PM状态
echo auto > /sys/devices/pci0000:b4/0000:b4:02.0/power/control
//使能SAS控制器自动runtime PM
echo 2000 > /sys/devices/pci0000:b4/0000:b4:02.0/power/autosuspend_delay_ms
//设置SAS控制器进入runtime PM状态前设备空闲的时间为2s

```

