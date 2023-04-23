
# openEuler WayCa SIG 鲲鹏调测工具 hikptool工具介绍


##  hikptool功能描述

  hikptool是用于鲲鹏芯片上增强带内信息收集和提升问题定位能力的linux用户态工具

  hikptool支持关键特性如下：
- 支持SAS模块信息查询功能：包含：CQ/DQ、链路误码、AXI、错误统计、设备SAS_ADDR、ITCT表项信息。
- 支持SERDES模块信息查询功能：包含：链路关键信息和关键寄存器dump。
- 支持CXL模块信息查询功能：包含：IO link/MEM link、CPA/DL寄存器、MEMAR_BAR、RCRB信息。
- 支持SATA模块信息查询功能：dump Global和port DFX寄存器。
- 支持RoH模块信息查询功能：包含：MAC模式、收发包、信用证、协议头信息、MAC CAM表项。
- 支持RoCE模块信息查询功能：包含：流控算法参数、收发包、内部子模块状态信息。
- 支持SoCIP模块信息查询功能：关键子模块寄存器dump。
- 支持NIC模块信息查询功能：包含：关键子模块寄存器dump，固件日志查询，PF硬件资源信息，端口建链信息，端口光模块信息，GRO信息，FEC异常统计，以及队列、混杂、流表、RSS算法、流控、转发表信息。
- 支持PCIE模块信息查询功能：包含：TRACE、链路状态、PCIE port划分信息查询。

__注意：只有root用户或者获得root授权的普通用户才能使用hikptool工具。__


##  特性详解

### 支持SAS模块信息查询功能
- 特性详解：
1. 支持查询SAS CQ指针和CQE信息：打印16个读写指针信息，CQE个数和聚合参数。
2. 支持查询SAS DQ指针和DQE信息：打印16个读写指针信息，已下发及未完成的DQE个数。
3. 支持查询指定SAS控制器的dev link，以及关SMMU时的设备配置信息。
4. 支持查询关SMMU时指定SAS控制器上的dqe配置信息。
5. 支持dump SAS的全局dfx、指定phy的dfx、axi dfx信息。
6. 支持查询指定SAS控制器的phy误码信息。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/sas


### 支持SERDES模块信息查询功能
- 特性详解：
1. 支持dump关键寄存器，支持cs、ds、subctrl三个子模块，dump时需要指定cpu和lane。
2. 支持查询serdes链路关键信息，支持显示简略和详细信息，简略信息包括：pll，pn状态，上电状态，速率，模式等。详细信息包括：FFE，CTEL，DFE，四点数字眼图等。dump时同样需要指定cpu和lane。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/serdes


### 支持CXL模块信息查询功能
- 特性详解：
1. 支持查询CXL_CPA：错误信息、关键配置、mmrg窗口信息和dump关键寄存器。
2. 支持查询CXL_DL：FSM状态机、DFX统计计数、错误信息和dump关键寄存器。
3. 支持查询CXL_MEMBAR：错误信息和dump关键寄存器。
4. 支持查询CXL_RCRB：link状态和dump关键寄存器。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/cxl


### 支持SATA模块信息查询功能
- 特性详解：
1. 支持dump全局dfx寄存器
2. 支持dump指定SATA port的dfx寄存器
   dump指定对应的cpu和die。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/sata


### 支持RoH模块信息查询功能
- 特性详解：
1. 支持查询指定RoH网口的状态信息
2. 支持查询指定RoH网口的反压状态。
3. 支持查询指定RoH网口的收发包统计信息。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/net/roh


### 支持RoCE模块信息查询功能
- 特性详解：
1. 支持查询CAEP模块相关统计及状态寄存器信息。
2. 支持查询GMV模块相关统计及状态寄存器信息。
3. 支持查询和清除MDB模块相关统计及状态寄存器信息。
4. 支持查询和清除RDMA报文统计信息。
5. 支持查询QMM模块相关统计及状态寄存器信息。
6. 支持查询和清除SCC模块相关统计及状态寄存器信息。
7. 支持查询和清除TIMER模块相关统计及状态寄存器信息。
8. 支持查询TRP、TSP、TGP_TMP模块相关统计及状态寄存器信息。
9. 支持查询和清除TDP模块相关统计及状态寄存器信息。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/net/roce


### 支持SoCIP模块信息查询功能
- 特性详解：
支持dump关键子模块寄存器：包括：I2C、GPIO、SPI、SFC、BTC。dump时需要指定cpu、die，每个子模块下支持不同的控制器ID，例如I2C模块可以dump不同I2C总线的寄存器。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/socip


### 支持NIC模块信息查询功能
- 特性详解：
1. 支持dump指定网口的MAC关键寄存器。
2. 支持查询指定网口的MAC链路信息。
3. 支持查询指定网口的光模块信息。
4. 支持查询指定模块的dfx统计和状态信息。
5. 支持查询指定网口的流表的硬件信息，规则和counter计数。
6. 支持查询FEC错误信息。
7. 支持查询GRO信息。
8. 支持查询所有或者指定网口的基本硬件信息。
9. 支持查询网口固件日志。
10. 支持查询指定网口的转发表、流控、队列、RSS信息。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/net/nic


### 支持PCIE模块信息查询功能
- 特性详解：
1. 支持dump关键寄存器，包括Global层和port层。
2. 支持查询pcie chip端口划分信息，查询和清除本地误码统计。
3. 支持基于白名单的寄存器读写。
4. 支持查询、清除trace。
5. 支持设置指定pcie port的trace fifo的收集模式。
6. 支持查询指定pcie port的链路状态信息。

- 支持版本：openEuler 22.03 lts sp1

- 代码路径
  https://gitee.com/openeuler/hikptool/tree/master/pcie


##  OS使能
- 编译依赖：cmake、make、gcc
- 运行依赖：glibc
- src包路径：https://gitee.com/src-openeuler/hikptool/tree/openEuler-22.03-LTS-SP1-release/


##  安装使用
- 1. 安装
    - yum 安装
  直接使用yum install hikptool安装即可

    - rpm包安装
  使用src包的路径源码可以构建出rpm包，再使用rpm进行安装，即可使用。
  rpm -ivh hikptool-xxx.aarch64.rpm

- 2. 运行
  hikptool -h，得到如下回显，说明工具正常运行（**需要root权限**）
>      Usage: hikptool <major_cmd> [option]
>
>        -h, --help    show help information
>        -v, --version show version information
>
>      Major Commands:
>
>        cxl_cpa                  cxl_cpa maininfo
>        cxl_dl                   cxl_dl maininfo
>        cxl_membar               cxl_membar maininfo
>        cxl_rcrb                 cxl_rcrb maininfo
>        nic_dfx                  dump dfx info of hardware
>        nic_fd                   dump fd info of nic!
>        nic_fec                  dump fec info of nic!
>        nic_gro                  dump gro info of nic!
>        nic_info                 show basic info of network!
>        nic_log                  dump m7 log info.
>        nic_mac                  dump mac module reg information
>        nic_port                 query nic port information
>        nic_ppp                  dump ppp info of nic!
>        nic_qos                  show qos info of nic!
>        nic_queue                dump queue info of nic!
>        nic_rss                  show rss info of nic!
>        nic_xsfp                 query port optical module information
>        pcie_dumpreg             pcie dump important regs for problem location
>        pcie_info                pcie information
>        pcie_regrd               pcie reg read for problem location
>        pcie_trace               pcie ltssm trace
>        roce_caep                get roce_caep registers information
>        roce_gmv                 get roce_gmv registers information
>        roce_mdb                 get or clear roce_mdb registers information
>        roce_pkt                 get or clear roce_pkt registers information
>        roce_qmm                 get roce_qmm registers information
>        roce_scc                 get or clear roce_scc registers information
>        roce_timer               get or clear roce_timer registers information
>        roce_trp                 get roce_trp registers information
>        roce_tsp                 get or clear roce_tsp registers information
>        roh_mac                  get roh mac information
>        roh_show_bp              get roh bp information
>        roh_show_mib             get roh mac mib information
>        sas_anacq                sas analysis cq queue
>        sas_anadq                sas analysis dq queue
>        sas_dev                  sas device information
>        sas_dqe                  sas dqe information
>        sas_dump                 sas reg dump
>        sas_errcode              sas read error code
>        sata_dump                sata reg dump
>        serdes_dump              serdes_dump cmd
>        serdes_info              serdes_info cmd
>        socip_dumpreg            Dump SoCIP registers

  简单测试：
  运行hikptool nic_info -i 网口名，可以查看当前网口的基本信息。预期得到类似如下回显:
  >     Current function: pf0
  >              pf mode:         ARM
  >              bdf id:          0000:xx:00.0
  >              mac id:          0
  >              mac type:        ETH
  >              func_num:        5
  >              tqp_num:         20
  >              pf_cap_flag:     0x0
  >              dev name:        ethxx
  >
  >     Current die(chipxx-diexx) info:
  >        revision id: 0x30
  >        mac mode: 0
  >        pf number: 2
  >        pf's capability flag: 0x2
  >        pf's attributes and capabilities:
  >        pf id:          pf0     pf1
  >        pf mode:        ARM     ARM
  >        mac id:         macxx    macxx
  >        mac type:       ETH     ETH
  >        func num:       5       5
  >        tqp num:        20      20


- 3. 卸载
  rpm -e hikptool-xxx.aarch64

##  问题交流
  https://gitee.com/openeuler/hikptool/issues
