# openEuler WayCa SIG 板载网卡驱动(内核态)模块介绍

## HNS3内核网卡驱动简介

- HNS3 driver是适配鲲鹏920系列处理器网络控制器的驱动模块，提供了完备的ethernet网卡功能。
  基础特性已在linux kernel 5.5完成支持。新增重要特性为linux kernel 5.5及以上支持的特性, 涵盖特性简介和社区信息。

- **源码获取路径**
    linux kernel 仓库：[https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)
    openeuler 仓库：[https://gitee.com/openeuler/kernel.git](https://gitee.com/openeuler/kernel.git)
    源码目录：**drivers/net/ethernet/hisilicon/hns3**

- **内核相关配置**
  - HNS3模块涉及以下5个Config选项：

|  配置项  | 功能  |
|  :----  | :----  |
| CONFIG_HNS3  | 必选，基础配置 |
| CONFIG_HNS3_ENET  | 必选，基础配置 |
| CONFIG_HNS3_HCLGE  | 必选，基础配置 |
| CONFIG_HNS3_HCLGEVF  | 可选，HNS3提供的虚拟网卡功能 |
| CONFIG_HNS3_DCB  | 可选，DCB功能 |

  - 部分特性依赖的内核Config选项

|  配置项  | 功能  |
|  :----  | :----  |
| CONFIG_RFS_ACCEL  | 可选，aRFS功能 |
| CONFIG_PCI_IOV  | 可选，SRIOV功能 |
| CONFIG_VLAN_8021Q  | 可选，802.1Q VLAN卸载功能 |

- **基础版本特性列表**
  - **支持多种接口形态**
    - 支持光口、电口、背板多种形态
    - 支持10M/100M/1000M/10G/25G/40G/50G/100GbE多种速率
    - 支持光模块自适应
    - 支持端口自协商
    - 支持端口速率双工配置
  - **支持CPU卸载**
    - 支持TX/RX校验和卸载
    - 支持TSO/GRO卸载
    - 支持VLAN TAG卸载
    - 支持RSS
    - 支持aRFS
    - 支持XPS
    - 支持流表
  - **支持QoS**
    - 支持ETS
    - 支持PFC
    - 支持普通流控
  - **支持vSwitch**
    - 支持MAC过滤
    - 支持VLAN过滤
  - **支持基本配置**
    - 支持配置mtu
    - 支持配置mac地址
    - 支持配置混杂模式
    - 支持中断聚合
    - 支持配置队列数
    - 支持配置队列深度
  - **支持SR-IOV**
    - 最大支持248个VF，单PF最大支持128个VF
    - 支持配置VF mac地址
    - 支持配置VF link状态
    - 支持配置VF防欺诈
    - 支持配置VF trust
    - 支持配置VF 带宽
    - 支持配置VF VLAN
  - **支持故障检测和恢复处理**
    - 支持RAS
    - 支持异常中断处理
    - 支持环回自检
- **新增重要特性包括**
  - 支持查询光模块 eeprom 信息
  - 支持UDP GSO卸载
  - 支持PTP(高精度时钟协议)
  - 支持硬件tc 卸载
  - 支持各层级复位
  - 支持 rx pagepool
  - 支持DSCP
  - 支持PMU


### 特性1: 支持查询光模块eeprom信息

- 特性介绍
  光口上可能会插入不同类型的光模块介质，PF可以向IMP查询介质模块的eeprom信息，供上层应用读取和解析。可以用于识别检查设备介质是否被更换或故障定位。

- 内核相关配置
  无

- 软件接口
  可通过ethtool -m <pf name> [raw on|off] [hex on|off] [offset N] [length N]命令，来查询当前介质模块的eeprom信息。显示形式由参数决定，可以为二进制、十六进制、或解析后的文本形式。可自定义查询的偏移和长度。
  驱动对外接口
	<code>int hns3_get_module_eeprom(struct net_device *netdev,
struct ethtool_eeprom *ee, u8 *data)</code>

- 涉及代码与使能

linux 主线合入信息

| COMMITID | SUBJECT | TAG |
| :----: | :----: | :----: |
| cb10228d234c  | [net: hns3: adds support for reading module eeprom info](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cb10228d234c49e2035bfce7bdb42c29e1049c5c) | kernel v5.7-rc2 |

openEuler  OLK-5.10使能信息

| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED(Y/N) |
| :----: | :----: | :----: |
| cb10228d234c | net: hns3: adds support for reading module eeprom info | Y |

### 特性2: 支持UDP GSO卸载

- 特性介绍
  UDP segment offload(UDP GSO)，类似TSO（TCP GSO），使得应用层可能生成比实际MTU更多的网络报文, 实际大小受底层网络和链路层协议限制。网络协议栈能够将大块buffer推送至网卡，由网卡执行分片工作，这样减轻了CPU的负荷。鲲鹏920系列处理器上板载网卡硬件提供了对UDP报文的分片功能。此功能PF/VF可以单独使能，支持UDPv4/UDPv6 以及隧道报文。

- 内核相关配置
无

- 软件接口
	驱动要声明支持NETIF_F_GSO_UDP_L4 feature, 方可使能此功能。可通过ethtool -K <devname> tx-udp-segmentation on/off 命令控制该功能的开启和关闭。

	UDP协议并不是面向字节流的，如果要支持UDP GSO，它需要通过发送缓冲区的大小隐式地表示了数据报的边界。应用程序必须在send调用时**使用额外参数明确标示这个数据报的gso大小**（socket option SOL_UDP/UDP_SEGMENT 或control message，使用udpgso_bench_tx/rx工具测试时，如果不指定MSS，将由协议栈分段，指定时由硬件分段）。内核构造较大的数据报，并连同相应元数据传递给UDP和IP层。网络设备或GSO层接收大数据报，将有效负载分成gso大小的段，然后复制头部。

- 涉及代码与使能

linux 主线合入信息

|  COMMITID   | SUBJECT  |  TAG |
|  :----:  | :----:  | :----: |
| 507e46ae26ea | [net: hns3: add getting capabilities of gro offload and fd from firmware](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=507e46ae26ea) | v5.18-rc7 |

openEuler  OLK-5.10使能信息

| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED(Y/N) |
| :----: | :----: | :----: |
| 72b042b4288c | net: hns3: add getting capabilities of gro offload and fd from firmware | Y |

### 特性3: 支持PTP
- 特性介绍
  PTP(precision time protocol)，精确时钟协议，又称IEEE 1588(v2)协议，用于对分布式系统提供亚微秒级别的时钟同步功能。PTP作为一种主从同步系统，在系统过程中，主时钟周期性发布PTP时间同步协议及时间信息，从时钟端口接收主时钟端口发来的时间戳信息，系统据此计算出主从线路时间延迟及主从时间差，并利用时间差调整本地时间，使从设备时间保持与主设备时间一致的频率和相位。

- 内核相关配置

|  配置项  | 功能  |
|  :----  | :----  |
| CONFIG_PTP_1588_CLOCK  | 可选，ptp功能 |

- 软件接口
  可通过ethtool -T <pf_name> 查询当前物理设备支持的时钟能力。
  驱动侧实现ndo_do_ioctl接口和ethtool_ops. get_ts_info接口，对应SIOCGHWTSTAMP，返回硬件时间戳信息配置，包括hwtstamp_tx_types和hwtstamp_rx_filters。
  Linux提供了标准的ptp配置工具，需要安装linuxptp，详见ptp4l/ptp2sys，使用方法可以参考[redhat的帮助文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/s1-starting_ptp4l)。

- 涉及代码与使能

linux 主线合入信息

| COMMITID | SUBJECT | TAG |
| :----: | :----: | :----: |
| 0bf5eb788512 | [net: hns3: add support for PTP](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0bf5eb788512) | v5.13-rc3 |
| b34c157f0cdd | [net: hns3: add debugfs support for ptp info](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b34c157f0cdd) | v5.13-rc3 |
| 8373cd38a888 | [net: hns3: change the method of obtaining default ptp cycle](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8373cd38a888) | v5.13-rc3 |

openEuler  OLK-5.10使能信息

| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED(Y/N) |
| :----: | :----: | :----: |
| 56acfb181aba | net: hns3: add support for PTP | Y |
| 0e093eb72d95 | net: hns3: add debugfs support for ptp info | Y |
| f61992ad87f2 | net: hns3: change the method of obtaining default ptp cycle | Y |

### 特性4: 支持硬件tc 卸载
- 特性介绍
  基线版本的驱动中，只支持了TC_MQPRIO_MODE_DCB模式，卸载了用户优先级和TC之间的映射关系，新驱动对此进行了增强，通过指定channel模式和hw配置全卸载，即TC_MQPRIO_MODE_CHANNEL模式，可以进一步为每个TC指定队列(各个TC可以不同，但受硬件限制，要求都是2的指数幂)。

- 内核相关配置

|  配置项  | 功能  |
|  :----  | :----  |
| CONFIG_NET_SCH_MQPRIO  | 可选，硬件tc卸载功能 |

- 软件接口
可通过tc qdisc命令进行配置，参数如下：
<code>tc qdisc add dev <pf name> root mqprio  num_tc <tc num> map <P0 P1 P2...  > queues count1@offset1 count2@offset2 ...   hw 1 mode channel</code>
如命令<code>tc qdisc add eth1 root mqprio num_tc 4 map 0 2 4 6 queues 4@0 8@4 8@12 4@20 hw 1 mode channel</code>
表示为eth1配置使能4个tc，优先级2映射tc1, 优先级4映射tc2, 优先级6映射tc3，其余优先级映射tc0, tc0绑定队列0-3，tc1绑定队列4-11，tc2绑定队列12-19，tc3绑定队列20-23。
清除配置命令 <code>tc qdisc del dev eth1 root</code>
查询配置命令 <code>tc filter show dev eth1 parent ffff:</code>

- 涉及代码与使能

linux 主线合入信息

| COMMITID | SUBJECT | TAG |
| :----: | :----: | :----: |
| 35244430d624 | [net: hns3: refine the struct hane3_tc_info](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=35244430d624) | v5.10-rc6 |
| 5a5c90917467 | [net: hns3: add support for tc mqprio offload](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5a5c90917467) | v5.10-rc6 |
| ae9e492a3664 | [net: hns3: remove redundant client_setup_tc handle](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ae9e492a3664) | v5.11-rc6 |
| 0472e95ffeac | [net: hns3: fix mixed flag HCLGE_FLAG_MQPRIO_ENABLE and HCLGE_FLAG_DCB_ENABLE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0472e95ffeac) | v5.15-rc2 |
| 161ad669e6c2 | [net: hns3: reconstruct function hclge_ets_validate()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=161ad669e6c2) | v5.14-rc7 |
| d82650be60ee | [net: hns3: don't rollback when destroy mqprio fail](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d82650be60ee) | v5.15-rc2 |
| a8e76fefe3de | [net: hns3: remove tc enable checking](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a8e76fefe3de) | v5.15-rc2 |

openEuler  OLK-5.10使能信息

| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED(Y/N) |
| :----: | :----: | :----: |
| e887fccc775f | net: hns3: refine the struct hane3_tc_info | Y |
| 46961be39152 | net: hns3: add support for tc mqprio offload | Y |
| bdb36e4c1180 | net: hns3: remove redundant client_setup_tc handle | Y |
| 9bb0177d462e | net: hns3: fix mixed flag HCLGE_FLAG_MQPRIO_ENABLE and HCLGE_FLAG_DCB_ENABLE | Y |
| 688795be38d6 | net: hns3: reconstruct function hclge_ets_validate | Y |
| ba250d23d0c6 | net: hns3: don't rollback when destroy mqprio fail | Y |
| dad19338e6a5 | net: hns3: remove tc enable checking | Y |

### 特性5: 支持各层级复位
- 特性介绍
  - 支持function复位
  function复位包含PF复位和VF复位，PF产生了复位，其管理的所有VF也要一起复位。当VF复位时，仅需复位VF自身。复位前后，网卡的配置保持一致。
  - PF支持global复位
  global复位是整个DIE上所有PF、VF复位。通常是在硬件发生严重故障时发生，需要通过global复位重置更多硬件模块才可能恢复。
  - PF支持IMP复位
  IMP复位不仅复位整个DIE上所有PF、VF，同时还复位IMP固件。在硬件处理上，IMP复位重置的硬件模块和global复位一样。

  优先级上 IMP复位 > Global复位 >  Function复位，同时产生多个复位源时，优先执行高优先级复位

  - 支持FLR复位
  FLR复位是由PCIE框架下发的复位请求，与function复位类似，区别是触发调用的接口不同。
- 内核相关配置
  FLR依赖PCIe的capbility中置位FLR能力，为可选项

- 复位触发源
  - 异常中断
   当芯片检测到NIC硬件异常时，会上报异常中断，NIC驱动会从IMP查询异常中断源及建议恢复方式，包括仅记录日志、VF复位、PF复位、Global复位和IMP复位。
  - tx timeout
   tx timeout是协议栈的发包超时检测机制，默认单个报文5秒钟未发送完成，则会调用NIC驱动的发包超时处理接口。默认处理为function复位。
   - pci框架触发
   解绑或绑定pci设备时，会触发FLR复位
   AER故障恢复时触发复位
  - 用户接口触发
    通过ethtool工具可以触发PF复位、VF复位、global复位和imp复位
    echo命令触发flr复位

- 软件接口
  - 支持function复位
  用户可以通过ethtool --reset ethX dedicated对网口发起function复位。

  - 支持FLR复位
  用户可通过“echo 1 > /sys/class/net/ethx/device/reset”命令直接发起FLR。FLR依赖PCIe的capbility中置位FLR能力，为可选项。需避免FLR复位与其他复位并发。

  - PF支持global复位
  用户可以通过ethtool --reset ethx all发起global复位。

  - PF支持IMP复位
  用户可以通过ethtool --reset ethx mgmt发起IMP复位

- 涉及代码与使能

linux 主线合入信息

| COMMITID | SUBJECT | TAG |
|  :----:  | :----:  | :----: |
| ddccc5e368a3 | [net: hns3: add support for triggering reset by ethtool](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ddccc5e368a3) | v5.14-rc4 |
| 82229c4dbb8a | [net: hns3: fix incorrect components info of ethtool --reset command](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=82229c4dbb8a) | v5.16-rc3 |

openEuler  OLK-5.10使能信息

| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED(Y/N) |
| :----: | :----: | :----: |
| 3d489ec0f3a4 | net: hns3: add support for triggering reset by ethtool | Y |
| 238a3d250c35 | net: hns3: fix incorrect components info of ethtool --reset command | Y |

### 特性6: 支持DSCP

- 特性介绍
  DSCP由RFC2474定义，它重新命名了IPv4报头中TOS字节和IPv6报头中数据类（TrafficClass）字节，新的名字称为DS字段（Differentiated ServicesField）。该字段的作用没有变，仍然被QoS工具用来标记数据。DSCP使用高6bit，最低2bit特不用，DSCP范围为0~63。驱动新增配置DSCP和TC的映射。

- 内核相关配置
无

- 软件接口
  驱动下发优先级和TC映射命令给IMP时，IMP会将芯片切到priority映射TC模式。驱动下发DSCP和TC映射命令给IMP时，IMP会将芯片切到DSCP映射TC模式。当前dcb标准工具和协议栈支持配置DSCP和priority的映射，驱动新增实现配置DSCP和priority映射的接口，并在驱动中转换成DSCP和TC的映射。
  在debugfs的tm目录增加qos_dscp_map文件，用户cat此文件时将DSCP与priority、TC的映射关系打印出来（仅打印用户已配置的DSCP），以及当前TC映射的模式。

- 涉及代码与使能

linux 主线合入信息

| COMMITID | SUBJECT | TAG |
| :----: | :----: | :----: |
| dfea275e06c2 | [net: hns3: optimize converting dscp to priority process of hns3_nic_select_queue()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dfea275e06c2) | v5.18-rc7 |
| fddc02eb583a | [net: hns3: debugfs add dump dscp map info](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fddc02eb583a) | v5.18-rc7 |
| f6e32724ca13 | [net: hns3: support ndo_select_queue()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f6e32724ca13) | v5.18-rc7 |
| 0ba22bcb222d | [net: hns3: add support config dscp map to tc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0ba22bcb222d) | v5.18-rc7 |
| cfdcb075048c | [net: hns3: fix get wrong value of function hclge_get_dscp_prio()](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cfdcb075048c) | v5.18-rc7 |

openEuler  OLK-5.10使能信息

|  COMMITID   | SUBJECT  |  openeuler OLK-5.10 ENABLED(Y/N) |
|  :----:  | :----:  | :----: |
| 3f0019a0e172  | net: hns3: optimize converting dscp to priority process of hns3_nic_select_queue() | Y |
| da7620f6b113  | net: hns3: debugfs add dump dscp map info | Y |
| 757d2e9b7454  | net: hns3: support ndo_select_queue() | Y |
| 7a1313b356de  | net: hns3: add support config dscp map to tc | Y |
| fb96fb425027  | net: hns3: fix get wrong value of function hclge_get_dscp_prio() | Y |

### 特性7: 支持PMU
- 特性介绍
在分析端到端的性能瓶颈时，需要分析软件和硬件每个处理阶段的耗时。硬件在鲲鹏920系列处理器上为板载网卡提供了IO PMU硬件模块，NIC驱动以实现一个perf驱动的方式将该模块适配到内核的perf框架中，并提供合理的用户接口给perf用户。用户可以用IO PMU模块来收集在实际业务场景下的硬件关键点的耗时点。

- 内核相关配置

|  配置项  | 功能  |
|  :----  | :----  |
| CONFIG_HNS3_PMU  | 可选，IO PMU功能 |

- 软件接口
功能使能需要加载模块hns3_pmu.ko。设备路径名/sys/devices/hns3_pmu_sicl_\<sicl_id> 。可通过命令进行事件查询perf stat -g -e hns3_pmu_sicl_0/config=0x00002,global=1/ -e hns3_pmu_sicl_0/config=0x10002,global=1/ -I 1000, 其中config=0xXXXXX代表查询事件。查询显示结果为两列查询事件统计信息, 一列是counter数据,主要事件有: 字节数，包个数，时钟数，中断个数; 另一列是ext_counter数据, 主要事件有: 包个数，时钟数。对数据进行统计计算可以计算出事件的检测值, 计算公式为result = counter ÷ ext_counter

- 涉及代码与使能

|  COMMITID   | SUBJECT  | TAG |
|  :----:  | :----:  | :----: |
| aaaee7b55c9e  | [docs: perf: Include hns3-pmu.rst in toctree to fix 'htmldocs' WARNING](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aaaee7b55c9e) | v5.18-rc7 |
| 66637ab137b4  | [drivers/perf: hisi: add driver for HNS3 PMU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=66637ab137b4) | v5.18-rc7 |
| 39915b6b5fc2  | [drivers/perf: hisi: Add description for HNS3 PMU driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=39915b6b5fc2) | v5.18-rc7 |

openEuler  OLK-5.10使能信息

| COMMITID | SUBJECT | openeuler OLK-5.10 ENABLED(Y/N) |
| :----: | :----: | :----: |
| 494f4da5dbc1 | docs: perf: Include hns3-pmu.rst in toctree to fix 'htmldocs' WARNING | Y |
| 5766c797a02a | drivers/perf: hisi: add driver for HNS3 PMU | Y |
| 36b83702e3cc | drivers/perf: hisi: Add description for HNS3 PMU driver | Y |
