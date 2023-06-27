# openEuler WayCa SIG 鲲鹏高速网络 RoCE驱动模块介绍

[TOC]

## HNS RoCE驱动简介

HNS RoCE驱动是适配鲲鹏920系列处理器板载网卡RoCE控制器的驱动模块。RoCE(RDMA over Converged Ethernet)，是由IBTA标准化组织定义的一种在以太网上采用RDMA技术的网络互联技术，可以提供低延迟和高性能的网络通信能力，适用于HPC，金融以及存储等需要高性能加速的场景。

### 源码获取路径

- **内核态**：

  linux kernel 仓库：http://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git

  openeuler/kernel 仓库：https://gitee.com/openeuler/kernel.git

  源码目录：drivers/infiniband/hw/hns

- **用户态**：

  linux/rdma-core 仓库：https://github.com/linux-rdma/rdma-core

  src-openeuler/rdma-core仓库：https://gitee.com/src-openeuler/rdma-core

  源码目录：providers/hns

### 内核相关配置

- RoCE模块涉及的Config选项

  | 配置项                      | 功能                         |
  | --------------------------- | ---------------------------- |
  | CONFIG_HNS3                 | 必选，基础配置               |
  | CONFIG_HNS3_ENET            | 必选，基础配置               |
  | CONFIG_HNS3_HCLGE           | 必选，基础配置               |
  | CONFIG_HNS3_HCLGEVF         | 可选，HNS3提供的虚拟网卡功能 |
  | CONFIG_INFINIBAND_HNS       | 必选，基础配置               |
  | CONFIG_INFINIBAND_HNS_HIP08 | 必选，基础配置               |

- 部分特性依赖的内核Config选项

  | 配置项                                | 功能                               |
  | ------------------------------------- | ---------------------------------- |
  | CONFIG_PCI_IOV                        | 可选，SRIOV功能                    |
  | CONFIG_VLAN_8021Q                     | 可选，802.1Q VLAN卸载功能          |
  | CONFIG_INFINIBAND                     | 必选，基础配置                     |
  | CONFIG_INFINIBAND_USER_ACCESS         | 必选，基础配置，支持RoCE用户态功能 |
  | CONFIG_INFINIBAND_USER_MAD            | 可选，支持用户态MAD报文            |
  | CONFIG_INFINIBAND_USER_MEM            | 必选，基础配置                     |
  | CONFIG_INFINIBAND_ADDR_TRANS          | 可选，支持CM功能                   |
  | CONFIG_INFINIBAND_ADDR_TRANS_CONFIGFS | 可选，支持CM功能                   |

## RoCE特性详解

### 特性1: 支持RoCE设备管理

- **特性详解**

  支持10G/25G/40G/50G/100G/200G 速率的RoCE设备。

  支持通过RDMA子系统查询对应RoCE设备。用户可进入/sys/class/infiniband目录下查看到对应RoCE设备。

  支持用户通过对RoCE驱动进行加载和卸载实现新增或者卸载RoCE设备。

  用户可通过`ibv_devinfo -d ${dev_name}`查看设备的基础信息。

- **软件接口**

  | 软件接口                                     | 描述                                                  |
  | -------------------------------------------- | ----------------------------------------------------- |
  | ibv_get_device_list()                        | 获取系统上可用的RDMA设备列表                          |
  | ibv_free_device_list()                       | 释放设备列表字符串                                    |
  | ibv_get_device_name()、ibv_get_device_guid() | 获得设备信息、名字和Global Unique Identifier (GUID)号 |
  | ibv_open_device()                            | 获得设备的句柄以做更多的操作                          |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |     TAG     |
  | :----------: | :----------------------------------------------------------: | :---------: |
  | dd74282df573 | [RDMA/hns: Initialize the PCI device for hip08 RoCE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dd74282df573) | kernel 4.15 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                      SUBJECT                       | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :------------------------------------------------: | :-----------------------------: |
  | dd74282df573 | RDMA/hns: Initialize the PCI device for hip08 RoCE |                Y                |
  
### 特性2: 支持 PD 管理

- **特性详解**

  PD (protection domain)是RDMA最基本的访问控制功能。通过PD，RDMA可以限制队列（QP）对每个MR的访问权限，防止未授权的访问。在创建QP以及MR时，必须要将QP和MR关联到一个PD，因此，用户必须至少创建一个PD才能使用verbs。

  HNS RoCE支持单芯片16M PD。

- **软件接口**

  | 软件接口                          | 描述    |
  | --------------------------------- | ------- |
  | ibv_alloc_pd()、ib_alloc_pd()     | 创建 PD |
  | ibv_dealloc_pd()、ib_dealloc_pd() | 销毁 PD |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 3a63c964eaa1 | [RDMA/hns: Update some attributes of the RoCE device](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3a63c964eaa1) | kernel 4.20-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                       SUBJECT                       | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :-------------------------------------------------: | :-----------------------------: |
  | 3a63c964eaa1 | RDMA/hns: Update some attributes of the RoCE device |                Y                |

### 特性3: 支持 MR 管理

RDMA的所有工作请求都基于MR(Memory Region)完成，每个MR都需要与一个PD相关联，并且拥有一对密钥（lkey、rkey）用来实现硬件的内存访问权限的管理。

#### 支持 MR 注册

- **特性详解**

  支持用户将指定内存注册为MR 。

  HNS RoCE支持单芯片1M MR，MR大小无限制；支持用户使用大页内存，驱动根据系统页面自适应4K~128M页表。

- **软件接口**

  | 软件接口                                     | 描述                             |
  | :------------------------------------------- | :------------------------------- |
  | ibv_reg_mr()、ibv_reg_mr_iova()              | 注册 MR                          |
  | ibv_dereg_mr()                               | 解注册该MR，释放内存             |
  | ib_reg_user_mr()、ib_dereg_mr_user()         | 注册 IB_MR_TYPE_USER 类型的MR    |
  | ib_alloc_mr()、ib_map_mr_sg()、ib_dereg_mr() | 注册 IB_MR_TYPE_MEM_REG 类型的MR |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | 9a4435375cd1 | [IB/hns: Add driver files for hns RoCE driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a4435375cd1) | kernel 4.9-rc1 |

  openeuler OLK-5.10 合入信息

  |   COMMITID   |                    SUBJECT                     | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :--------------------------------------------: | :-----------------------------: |
  | 9a4435375cd1 |  IB/hns: Add driver files for hns RoCE driver  |                Y                |
  | 9a9fa2f04d16 | RDMA/hns: Fix level-0 addressing for huge page |                Y                |
  | cf2f2029935c |   RDMA/hns: Support adaptive hopnum for MTR    |                Y                |
  | cfad301fb303 |  RDMA/hns: Support flexible pagesize for MTR   |                Y                |
  | 5be3bbe7f0e6 | RDMA/hns: Alloc MTR memory before alloc_mtt()  |                Y                |
  | 47968dd6159d |     RDMA/hns: Refactor mtr_init_buf_cfg()      |                Y                |
  | 893026a80118 |        RDMA/hns: Fix PBL page MTR find         |                Y                |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |            SUBJECT             |      TAG       |
  | :----------: | :----------------------------: | :------------: |
  | 887b78c80224 | libhns: Add initial main frame | rdma-core 12.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |            SUBJECT             | openeuler ENABLED(Y/N) |
  | :----------: | :----------------------------: | :--------------------: |
  | 887b78c80224 | libhns: Add initial main frame |           Y            |

#### 支持 MR 重注册（Rereg MR）

- **特性详解**

  重注册MR支持修改现有MR的属性。功能上等同于Deregister MR + Register MR，但是会进行驱动内部资源的复用，减小重复的资源分配释放的动作。注意， Rereg MR仅支持用户态，内核态可以通过FRMR完成类似功能。

- **软件接口**

  | 软件接口       | 描述                                                       |
  | -------------- | ---------------------------------------------------------- |
  | ibv_rereg_mr() | 重新设置该MR的所有属性，每个属性、域段的功能与注册MR时一致 |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                   SUBJECT                   |      TAG       |
  | :----------: | :-----------------------------------------: | :------------: |
  | d326d54fed3e | libhns: Add rereg mr interface in userspace | rdma-core 16.0 |

  openeuler/rdma 使能信息

  | COMMITID     | SUBJECT                                     | openeuler  ENABLED(Y/N) |
  | ------------ | ------------------------------------------- | ----------------------- |
  | d326d54fed3e | libhns: Add rereg mr interface in userspace | Y                       |

#### 支持快速注册 MR（FRMR）

- **特性详解**

  FRMR(Fast register Physical memory region)，允许特权用户（内核态用户）通过post send IB_MR_TYPE_MEM_REG类型的WR的方式对处于free状态的MR进行快速注册。结合invalid类型的WR，可以实现MR的快速重注册，将MR的属性、内存等进行快速的修改。

  注意：最大支持512个页面的FRMR，要求SGL中的所有地址页对齐。

- **软件接口**

  | 软件接口       | 描述                             |
  | -------------- | -------------------------------- |
  | ib_alloc_mr()  | 创建 IB_MR_TYPE_MEM_REG 类型的MR |
  | ib_map_mr_sg() | 将 SGL 的DMA地址转换为驱动的PBL  |
  | ib_dereg_mr()  | 解注册MR，释放内存               |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 68a997c5d28c | [RDMA/hns: Add FRMR support for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=68a997c5d28c) | kernel 5.10-rc1 |
  | dad1f9802ece | [RDMA/hns: Configure capacity of hns device](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dad1f9802ece) | kernel 5.10-rc1 |
  | fd9e3679af8d | [RDMA/hns: Use new interface to write FRMR fields](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fd9e3679af8d) | kernel 5.10-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                     SUBJECT                      | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------: | :-----------------------------: |
  | 68a997c5d28c |       RDMA/hns: Add FRMR support for hip08       |                Y                |
  | 68a997c5d28c |    RDMA/hns: Configure capacity of hns device    |                Y                |
  | 79352749ea09 | RDMA/hns: Use new interface to write FRMR fields |                Y                |

### 特性4: 支持 CQ 管理

- **特性详解**

  CQ(completion queue)是WR完成状态的上报通道。RDMA中每个WR的完成状态由一个CQE（completion queue entry）描述。每个CQ可以管理多个CQE，可以支持一个或者多个SQ、RQ的完成状态的上报。

  HNS RoCE支持单芯片1M CQ，最大支持8M CQE。

- **软件接口**

  | 软件接口                                            | 描述         |
  | :-------------------------------------------------- | :----------- |
  | ibv_create_cq()、ibv_create_cq_ex()、ib_create_cq() | 创建CQ       |
  | ibv_modify_cq()、                                   | 修改CQ 配置  |
  | ibv_poll_cq()、ib_poll_cq()                         | 从CQ中获取WC |
  | ibv_destroy_cq()、ib_destroy_cq()                   | 销毁CQ       |


- **内核态涉及代码与使能**

  linux 主线信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 75c994e6943c | [RDMA/hns: Stop doorbell update while qp state error](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=75c994e6943c) | kernel 5.6-rc3  |
  | 0425e3e6e0c7 | [RDMA/hns: Support flush cqe for hip08 in kernel space](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0425e3e6e0c7) | kernel 4.19-rc1 |
  | b53742865e9f | [RDMA/hns: Delayed flush cqe process with workqueue](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b53742865e9f) | kernel 5.6-rc2  |
  | 60c3becfd1a1 | [RDMA/hns: Fix sg offset non-zero issue](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=60c3becfd1a1) | kernel 5.4-rc1  |
  | 0fc99566f6ee | [RDMA/hns: Use flush framework for the case in aeq](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0fc99566f6ee) | kernel 5.6-rc3  |
  | dc1d06e699b5 | [RDMA/hns: Remove unnecessary flush operation for workqueue](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dc1d06e699b5) |   kernel 5.13   |
  | c462a0242bd9 | [RDMA/hns: Encapsulate flushing CQE as a function](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c462a0242bd9) |   kernel 5.14   |
  | 783cf673b05e | [RDMA/hns: Fix memory corruption when allocating XRCDN](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=783cf673b05e) |   kernel 5.13   |
  | 495c24808ce7 | [RDMA/hns: Add XRC subtype in QPC and XRC type in SRQC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=495c24808ce7) |   kernel 5.13   |
  | 32548870d438 | [RDMA/hns: Add support for XRC on HIP09](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32548870d438) |   kernel 5.13   |

  openeuler OLK-5.10 使能信息

  | COMMITID     |                          SUBJECT                           | openeuler OLK-5.10 ENABLED(Y/N) |
  | ------------ | :--------------------------------------------------------: | :-----------------------------: |
  | 75c994e6943c |    RDMA/hns: Stop doorbell update while qp state error     |                Y                |
  | 0425e3e6e0c7 |   RDMA/hns: Support flush cqe for hip08 in kernel space    |                Y                |
  | b53742865e9f |     RDMA/hns: Delayed flush cqe process with workqueue     |                Y                |
  | 60c3becfd1a1 |           RDMA/hns: Fix sg offset non-zero issue           |                Y                |
  | 0fc99566f6ee |     RDMA/hns: Use flush framework for the case in aeq      |                Y                |
  | 1d500a3f452d | RDMA/hns: Remove unnecessary flush operation for workqueue |                Y                |
  | 66b1000f5715 |      RDMA/hns: Encapsulate flushing CQE as a function      |                Y                |
  | d3b33dffb51a |   RDMA/hns: Fix memory corruption when allocating XRCDN    |                Y                |
  | b44fac86ca3d |   RDMA/hns: Add XRC subtype in QPC and XRC type in SRQC    |                Y                |
  | ae394640bc89 |           RDMA/hns: Add support for XRC on HIP09           |                Y                |

- **用户态涉及代码与使能**

  linux/rdma-core 合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | e1726e934574 |      libhns: Support flush cqe for hip08 in user space       | rdma-core 28.0 |
  | 482eb44bc296 |       libhns: Not process return value of flushing cqe       | rdma-core 28.0 |
  | 366374ee6372 |   libhns: Update ibvqp->state in hns_roce_u_v2_modify_qp()   | rdma-core 28.0 |
  | f1a80cc3dfe2 |      libhns: Bugfix for flush cqe in case multi-process      | rdma-core 24.0 |
  | 4ed874a5cf30 |            libhns: Add support for XRC for HIP09             | rdma-core 24.0 |
  | 9e3df7578153 |              libhns: Support ibv_create_srq_ex               | rdma-core 24.0 |
  | 7c3937d93e7a | libhns: Avoid accessing NULL pointer when locking/unlocking CQ | rdma-core 24.0 |
  | d8596eff4eb4 |         libhns: Add support for creating extended CQ         | rdma-core 24.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :--------------------: |
  | e1726e934574 |      libhns: Support flush cqe for hip08 in user space       |           Y            |
  | 482eb44bc296 |       libhns: Not process return value of flushing cqe       |           Y            |
  | 366374ee6372 |   libhns: Update ibvqp->state in hns_roce_u_v2_modify_qp()   |           Y            |
  | f1a80cc3dfe2 |      libhns: Bugfix for flush cqe in case multi-process      |           Y            |
  | 4ed874a5cf30 |            libhns: Add support for XRC for HIP09             |           Y            |
  | 9e3df7578153 |              libhns: Support ibv_create_srq_ex               |           Y            |
  | 7c3937d93e7a | libhns: Avoid accessing NULL pointer when locking/unlocking CQ |           Y            |
  | d8596eff4eb4 |         libhns: Add support for creating extended CQ         |           Y            |

### 特性5: 支持 QP 管理

HNS RoCE支持单芯片1M个QP，支持最大32K的WR（Work Request）下发，每个工作请求支持64个SGE，单个SGE最大支持2G。支持256，512，1K，2K、4K规格的PMTU。支持RESET/INIT/RTR/RTS/ERR五种状态的迁移；支持RC、UD、XRC三种服务类型的QP。

#### 支持 RC 类型 的 QP

- **特性详解**

  HNS RoCE支持用户创建RoCEv1,RoCEv2 协议的RC类型的QP。

- **软件接口**

  | 软件接口                                            | 描述       |
  | :-------------------------------------------------- | :--------- |
  | ibv_create_qp()、ibv_create_qp_ex()、ib_create_qp() | 创建QP     |
  | ibv_modify_qp()                                     | 修改QP属性 |
  | ibv_query_qp()、ib_query_qp()                       | 查询QP信息 |
  | ibv_destroy_qp()、ib_destroy_qp()                   | 销毁QP     |

- **内核态涉及代码与使能**

  linux 主线合入信息

  | COMMITID     | SUBJECT                                                      | TAG         |
  | ------------ | ------------------------------------------------------------ | ----------- |
  | 926a01dc000d | [RDMA/hns: Add QP operations support for hip08 SoC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=926a01dc000d76df3f5b110dddcebfb517b8f57b) | kernel 4.15 |
  | 9a4435375cd1 | [IB/hns: Add driver files for hns RoCE driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a4435375cd151e07c0c38fa601b00115986091b) | kernel 4.9  |
  | 0f00571f9433 | [RDMA/hns: Use new SQ doorbell register for HIP09](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0f00571f94339fa27f592d157ccc0b909dc0625e) | kernel 5.13 |
  | 1bbd4380744f | [RDMA/hns: Create CQ with selected CQN for bank load balance](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1bbd4380744f637a759e0a7bb7d8d1c38282e0c3) | kernel 5.12 |
  | 71586dd20010 | [RDMA/hns: Create QP with selected QPN for bank load balance](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=71586dd2001087e89e344e2c7dcee6b4a53bb6de) | kernel 5.11 |
  | 9293d3fcb705 | [RDMA/hns: Use mutex instead of spinlock for ida allocation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9293d3fcb70583f2c786f04ca788af026b7c4c5c) | kernel 5.12 |
  | bfefae9f108d | [RDMA/hns: Add support for CQ stash](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bfefae9f108dfa62eb9c16c9e97086fddb4ece04) | kernel 5.10 |
  | f93c39bc9547 | [RDMA/hns: Add support for QP stash](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f93c39bc95472dae3b5de71da5c005f47ece3148) | kernel 5.10 |
  | 260f64a40198 | [RDMA/hns: Enable stash feature of HIP09](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=260f64a40198309008026447f7fda277a73ed8c3) | kernel 5.15 |

  openeuler OLK-5.10使能信息

  |   COMMITID   |                           SUBJECT                           | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :---------------------------------------------------------: | :-----------------------------: |
  | 926a01dc000d |      RDMA/hns: Add QP operations support for hip08 SoC      |                Y                |
  | 9a4435375cd1 |        IB/hns: Add driver files for hns RoCE driver         |                Y                |
  | e6ce4942dbde |      RDMA/hns: Use new SQ doorbell register for HIP09       |                Y                |
  | c746005b2800 | RDMA/hns: Create QP with selected QPN for bank load balance |                Y                |
  | 50c11848f7c8 | RDMA/hns: Create CQ with selected CQN for bank load balance |                Y                |
  | 56fbd51c835f | RDMA/hns: Use mutex instead of spinlock for ida allocation  |                Y                |
  | bf0eda201116 |             RDMA/hns: Add support for CQ stash              |                Y                |
  | 78d048bb6136 |             RDMA/hns: Add support for QP stash              |                Y                |
  | 01d06a58d6d3 |           RDMA/hns: Enable stash feature of HIP09           |                Y                |

- **用户态涉及代码与使能**

  linux/rdma-core 合入信息

  |   COMMITID   |            SUBJECT             |     TAG      |
  | :----------: | :----------------------------: | :----------: |
  | 887b78c80224 | libhns: Add initial main frame | rdma-core 12 |

  openeuler/rdma-core 使能信息

  |   COMMITID   |            SUBJECT             | openeuler ENABLED(Y/N) |
  | :----------: | :----------------------------: | :--------------------: |
  | 887b78c80224 | libhns: Add initial main frame |           Y            |


#### 支持 UD 类型的 QP 

- **特性详解**

  HNS RoCE支持用户创建RoCEv1,RoCEv2 协议的UD类型的QP。

- **软件接口**

  参考RC类型的QP。

- **内核态涉及代码与使能**

  linux 主线合入信息

  | COMMITID     | SUBJECT                                                      | TAG             |
  | ------------ | ------------------------------------------------------------ | --------------- |
  | d6d91e46210f | [RDMA/hns: Add support for configuring GMV table](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d6d91e46210f) | kernel 5.11-rc1 |
  | 32053e584e4a | [RDMA/hns: Add support for filling GMV table](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32053e584e4a) | kernel 5.11-rc1 |
  | 7af80c02c7b3 | [RDMA/hns: Fix double free of the pointer to TSQ/TPQ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7af80c02c7b3) | kernel 5.11-rc1 |
  | 98912ee82a0c | [RDMA/hns: Add support for QPC in size of 512 Bytes](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=98912ee82a0c) | kernel 5.10-rc1 |
  | 148f904c6f94 | [RDMA/hns: Remove the portn field in UD SQ WQE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=148f904c6f94) | kernel 5.11     |
  | 534c9bdb025b | [RDMA/hns: Simplify process of filling UD SQ WQE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=534c9bdb025b) | kernel 5.11     |
  | 66d86e529dd5 | [RDMA/hns: Add UD support for HIP09](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=66d86e529dd5) | kernel 5.11     |

  openeuler OLK-5.10使能信息

  |   COMMITID   |                       SUBJECT                       | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :-------------------------------------------------: | :-----------------------------: |
  | b762bf9b0516 |   RDMA/hns: Add support for configuring GMV table   |                Y                |
  | 115a32e39d13 |     RDMA/hns: Add support for filling GMV table     |                Y                |
  | 86d6d8ac4bb3 | RDMA/hns: Fix double free of the pointer to TSQ/TPQ |                Y                |
  | 98912ee82a0c | RDMA/hns: Add support for QPC in size of 512 Bytes  |                Y                |
  | 396b71fba124 |    RDMA/hns: Remove the portn field in UD SQ WQE    |                Y                |
  | 11703ad67174 |   RDMA/hns: Simplify process of filling UD SQ WQE   |                Y                |
  | 27eee3ced4ba |         RDMA/hns: Add UD support for HIP09          |                Y                |

- **用户态涉及代码与使能**

  参考RC类型的QP。

#### 支持 XRC 类型的 QP

- **特性详解**

  XRC(eXtended Reliable Connection)，目标是提高全连接数据模式的可伸缩性，同时限制双方所需数据结构的增长。

  XRC可以大大节省在大型集群中建立all-to-all进程连接所需的QP数量，它是RC、RD的混合。在请求方，它的行为与正常的RC SQ的行为相同；在接收方，存在与RD服务接收方的概念相似的数据结构，XRC在使用时，接收方向需要使用SRQ(SRQ具体用法见SRQ章节)，并且SRQ需要关联到对应的XRC域（XRCD）。

- **软件接口**

  创建XRC类型的QP的方式见RC部分说明。

  | 软件接口                                 | 描述     |
  | ---------------------------------------- | -------- |
  | ibv_open_xrcd()、ib_alloc_xrcd_user()    | 创建XRCD |
  | ibv_close_xrcd()、ib_dealloc_xrcd_user() | 销毁XRCD |

- **涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 32548870d438 | [RDMA/hns: Add support for XRC on HIP09](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32548870d438) | kernel 5.13-rc1 |
  | 783cf673b05e | [RDMA/hns: Fix memory corruption when allocating XRCDN](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=783cf673b05e) | kernel 5.13-rc1 |
  | 495c24808ce7 | [RDMA/hns: Add XRC subtype in QPC and XRC type in SRQC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=495c24808ce7) | kernel 5.12-rc1 |

  openeuler OLK-5.10使能信息

  |   COMMITID   |                        SUBJECT                        | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :---------------------------------------------------: | :-----------------------------: |
  | ae394640bc89 |        RDMA/hns: Add support for XRC on HIP09         |                Y                |
  | d3b33dffb51a | RDMA/hns: Fix memory corruption when allocating XRCDN |                Y                |
  | b44fac86ca3d | RDMA/hns: Add XRC subtype in QPC and XRC type in SRQC |                Y                |

### 特性6: 支持 SRQ 管理

- **特性详解**

  支持SRQ(shared receive queue)生命周期管理, SRQ的引入就是允许多个QP共享接收队列，以提高WQE利用率，减少内存消耗。支持全局1M数量规格，支持最大32K的WR list下发(SRQ队列的深度上限为32K)，每个工作请求支持64深度的SG list，单个SGE支持2G大小。

- **软件接口**

  | 软件接口                                                     | 描述            |
  | ------------------------------------------------------------ | --------------- |
  | ibv_create_srq()、ibv_create_srq_ex()、ibv_get_srq_num()、ibv_destroy_srq()、ib_create_srq_user()、ib_destroy_srq_user(） | 创建和销毁SRQ   |
  | ibv_modify_srq()、ib_modify_srq()                            | 修改SRQ属性     |
  | ibv_query_srq()、ib_query_srq()                              | 查询SRQ属性     |
  | ibv_post_srq_recv、ib_post_srq_recv()                        | 使用SRQ接收数据 |

- **内核态涉及代码与使能**

  linux 合入主线信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 495c24808ce7 | [RDMA/hns: Add XRC subtype in QPC and XRC type in SRQC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=495c24808ce7) |   kernel 5.13   |
  | 783cf673b05e | [RDMA/hns: Fix memory corruption when allocating XRCDN](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=783cf673b05e) |   kernel 5.13   |
  | 9d9d4ff78884 | [RDMA/hns: Update the kernel header file of hns](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d9d4ff78884) | kernel 5.1-rc1  |
  | 81fce6291d99 | [RDMA/hns: Add SRQ asynchronous event support](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=81fce6291d99) | kernel 4.20-rc6 |
  | c7bcb13442e1 | [RDMA/hns: Add SRQ support for hip08 kernel mode](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c7bcb13442e1) | kernel 4.20-rc6 |
  | 5c1f167af112 | [RDMA/hns: Init SRQ table for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5c1f167af112) | kernel 4.20-rc6 |
  | d16da11992d4 | [RDMA/hns: Eanble SRQ capacity for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d16da11992d4) | kernel 4.20-rc6 |
  
  openeuler OLK-5.10 使能信息

  |   COMMITID   |                        SUBJECT                        | openeuler  OLK-5.10 ENABLED(Y/N) |
  | :----------: | :---------------------------------------------------: | :------------------------------: |
  | 495c24808ce7 | RDMA/hns: Add XRC subtype in QPC and XRC type in SRQC |                Y                 |
  | d3b33dffb51a | RDMA/hns: Fix memory corruption when allocating XRCDN |                Y                 |
  | 9d9d4ff78884 |    RDMA/hns: Update the kernel header file of hns     |                Y                 |
  | 81fce6291d99 |     RDMA/hns: Add SRQ asynchronous event support      |                Y                 |
  | c7bcb13442e1 |    RDMA/hns: Add SRQ support for hip08 kernel mode    |                Y                 |
  | 5c1f167af112 |          RDMA/hns: Init SRQ table for hip08           |                Y                 |
  | d16da11992d4 |        RDMA/hns: Eanble SRQ capacity for hip08        |                Y                 |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                       SUBJECT                        |      TAG       |
  | :----------: | :--------------------------------------------------: | :------------: |
  | 9714b12735b2 |      libhns: Update poll cq for supporting srq       | rdma-core 23.0 |
  | 4332ae4947bb |       libhns: Add the verb for posting srqwqe        | rdma-core 23.0 |
  | 49db5b9ac356 |  libhns: Add destroy srq verbs for hip08 user mode   | rdma-core 23.0 |
  | 03aa74d63dd6 |    libhns: Add query srq verb for hip08 user mode    | rdma-core 23.0 |
  | 85ce85bb594a |   libhns: Add modify srq verb for hip08 user mode    | rdma-core 23.0 |
  | 22d53621d9b0 | libhns: Add verb of creating srq for hip08 user mode | rdma-core 23.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |                       SUBJECT                        | openeuler  ENABLED(Y/N) |
  | :----------: | :--------------------------------------------------: | :---------------------: |
  | 9714b12735b2 |      libhns: Update poll cq for supporting srq       |            Y            |
  | 4332ae4947bb |       libhns: Add the verb for posting srqwqe        |            Y            |
  | 49db5b9ac356 |  libhns: Add destroy srq verbs for hip08 user mode   |            Y            |
  | 03aa74d63dd6 |    libhns: Add query srq verb for hip08 user mode    |            Y            |
  | 85ce85bb594a |   libhns: Add modify srq verb for hip08 user mode    |            Y            |
  | 22d53621d9b0 | libhns: Add verb of creating srq for hip08 user mode |            Y            |

### 特性7: 支持建链

- **特性详解**

  HNS RoCE支持Socket建链和CM建链两种方式。

  Socket建链是指通信双方基于传统以太网Socket API完成参数协商，采用TCP协议进行交互, 因此采用Socket建链要求通信双方均支持TCP/IP协议。使用Socket建链时，参数协商的过程由用户自定义。

  CM建链是指通信双方基于RDMA协议CM报文（一种基于IP地址的UD报文协议）完成参数协商。RDMA框架提供了一套CM API，用户可以利用这套API完成通信双方QP的参数协商。 

- **软件接口**

  Socket建链由用户自定义，这里不再描述，下面介绍CM建链的接口：

  | 软件接口                        | 描述               |
  | ------------------------------- | ------------------ |
  | rdma_create_id()                | 创建RDMA CMD ID    |
  | rdma_bind_addr()                | 绑定需要监听的端口 |
  | rdma_listen()、rdma_connect()   | 用于监听和连接     |
  | rdma_get_cm_event()             | 等待CONNET事件     |
  | rdma_accept()、rdma_establish() | 完成连接           |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 8320deb88c03 | [RDMA/hns: Add enable judgement for UD vlan](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8320deb88c03) | kernel 4.20-rc1 |
  | 944e64093a63 | [RDMA/hns: Add CM of vlan device support](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=944e64093a63) | kernel 4.20-rc1 |
  | 6c1f08b347f6 | [RDMA/hns: Assign zero for pkey_index of wc in hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6c1f08b347f6) | kernel 4.16-rc1 |
  | 7bdee4158b37 | [RDMA/hns: Fill sq wqe context of ud type in hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7bdee4158b37) | kernel 4.16-rc1 |
  | 0fa95a9a7102 | [RDMA/hns: Add gsi qp support for modifying qp in hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0fa95a9a7102) | kernel 4.16-rc1 |
  | b66efc932067 | [RDMA/hns: Create gsi qp in hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b66efc932067) | kernel 4.16-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                        SUBJECT                         | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------: | :-----------------------------: |
  | 8320deb88c03 |       RDMA/hns: Add enable judgement for UD vlan       |                Y                |
  | 944e64093a63 |        RDMA/hns: Add CM of vlan device support         |                Y                |
  | 6c1f08b347f6 |  RDMA/hns: Assign zero for pkey_index of wc in hip08   |                Y                |
  | 7bdee4158b37 |   RDMA/hns: Fill sq wqe context of ud type in hip08    |                Y                |
  | 0fa95a9a7102 | RDMA/hns: Add gsi qp support for modifying qp in hip08 |                Y                |
  | b66efc932067 |            RDMA/hns: Create gsi qp in hip08            |                Y                |

### 特性8: 支持多种工作请求

- **特性详解**

  支持SEND操作。支持Send/Send With Immediate、Send with Invalid。

  支持RDMA WRITE操作。支持RDMA Write / RDMA Write With Immediate。

  支持RDMA READ操作。

  支持RDMA ATOMIC操作。Atomic Fetch and Add / Atomic Compare and Swap。

  支持最大32K的WR LIST下发，每个工作请求支持64个SG LIST，单个SGE最大支持2G。

- **软件接口**

  | 软件接口                         | 描述         |
  | -------------------------------- | ------------ |
  | ibv_post_send()、ibv_post_recv() | 下发工作请求 |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 9a4435375cd1 | [IB/hns: Add driver files for hns RoCE driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a4435375cd1) |   kernel 4.9    |
  | 384f88185112 | [RDMA/hns: Add atomic support](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=384f88185112) | kernel 4.20-rc1 |
  | ace1c5416b37 | [RDMA/hns: Set access flags of hip08 RoCE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ace1c5416b37) | kernel 4.16-rc1 |
  | d9581bf358c0 | [RDMA/hns: Bugfix for atomic operation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d9581bf358c0) | kernel 4.20-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                          SUBJECT                           | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :--------------------------------------------------------: | :-----------------------------: |
  | 9a4435375cd1 |        IB/hns: Add driver files for hns RoCE driver        |                Y                |
  | 84f88185112  |                RDMA/hns: Add atomic support                |                Y                |
  | ace1c5416b37 |          RDMA/hns: Set access flags of hip08 RoCE          |                Y                |
  | d9581bf358c0 |           RDMA/hns: Bugfix for atomic operation            |                Y                |
  | 555adba6b7a5 |     RDMA/hns: Fix a missing check of atomic wr length      |                Y                |
  | c0f88964aed6 | RDMA/hns: Fix inaccurate error label name in init instance |                Y                |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | 887b78c80224 |                libhns: Add initial main frame                | rdma-core 12.0 |
  | b8c02c0039c4 |           libhns: Add support for extended atomic            | rdma-core 28.0 |
  | d92b0f595439 |        libhns: Add atomic support for hip08 user mode        | rdma-core 21.0 |
  | a95f62b7e42f | libhns: Check number of extended sge when using extended atomic | rdma-core 29.0 |
  | 22edc93d4d17 |       libhns: Bugfix for atomic operation in user mode       | rdma-core 21.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :--------------------: |
  | 887b78c80224 |                libhns: Add initial main frame                |           Y            |
  | b8c02c0039c4 |           libhns: Add support for extended atomic            |           Y            |
  | d92b0f595439 |        libhns: Add atomic support for hip08 user mode        |           Y            |
  | a95f62b7e42f | libhns: Check number of extended sge when using extended atomic |           Y            |
  | 22edc93d4d17 |       libhns: Bugfix for atomic operation in user mode       |           Y            |

### 特性9: 支持inline

#### 支持SQ inline

- **特性详解**

  SQ inline是对于长度较小的消息的一种加速手段，RC、UD的WQE中都可以储存一定大小的inline data，最大支持到1024Bytes。仅用户态支持。

- **软件接口**

  使用`ibv_post_send()`接口时，显示设置ibv_send_wr的send_flags字段中的IBV_SEND_INLINE位为1, 则发送将走inline模式。

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 328d405b3d4c | [RDMA/hns: Intercept illegal RDMA operation when use inline data](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=328d405b3d4c) | kernel 4.18-rc1 |
  | 9eab614338cd | [RDMA/hns: Fix missing assignment of max_inline_data](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9eab614338cd) |   kernel 5.13   |
  | 30b707886aeb | [RDMA/hns: Support inline data in extented sge space for RC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=30b707886aeb) |   kernel 5.10   |
  | 0c5e259b06a8 | [RDMA/hns: Fix incorrect sge nums calculation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0c5e259b06a8) | kernel 6.2-rc1  |
  | 8eaa6f7d569b | [RDMA/hns: Fix ext_sge num error when post send](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8eaa6f7d569b) | kernel 6.2-rc1  |
  
  openeuler OLK-5.10 使能信息
  
  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | 328d405b3d4c | RDMA/hns: Intercept illegal RDMA operation when use inline data |                Y                |
  | e8638bded775 |     RDMA/hns: Fix missing assignment of max_inline_data      |                Y                |
  | 30b707886aeb |  RDMA/hns: Support inline data in extented sge space for RC  |                Y                |
  | 7f360a7e7771 |        RDMA/hns: Fix ext_sge num error when post send        |                Y                |
  | 345423493ae7 |            RDMA/hns: Fix the problem of sge nums             |                Y                |
  
- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | 328d405b3d4c | libhns: Intercept illegal RDMA operation when use inline data | rdma-core 19.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :--------------------: |
  | 328d405b3d4c | libhns: Intercept illegal RDMA operation when use inline data |           Y            |

#### 支持RQ inline

- **特性详解**

  支持RC服务类型，接收端支持将接收到的Send数据直接写入RQ WQE地址上报，从而减少硬件的内存访问，降低系统延迟。支持最大长度为1024B的RQ inline能力。仅用户态支持，仅支持单包接收，不能切包。RQ inline用户并不感知，驱动根据固件中的配置进行开启或者不开启该功能。

- **软件接口**

  NA

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 0009c2dbe8a4 | [RDMA/hns: Add rq inline data support for hip08 RoCE](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0009c2dbe8a4) | kernel 4.16-rc1 |
  | 2bb185c68bf4 | [RDMA/hns: Add compatibility handling for only support userspace rq inline](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2bb185c68bf4) | kernel 6.3-rc1  |
  

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | ecaaf1e26a37 |           RDMA/hns: Add rq inline flags judgement            |                Y                |
  | 0009c2dbe8a4 |     RDMA/hns: Add rq inline data support for hip08 RoCE      |                Y                |
  | a77029a8c7e4 | RDMA/hns: Fix the inconsistency between the rq inline bit and the community |                Y                |
  | c22b4e60d382 |         RDMA/hns: Fix the compatibility flag problem         |                Y                |

- **用户态涉及代码与使能**

  linux/rdma 主线合入信息

  |   COMMITID   |                           SUBJECT                           |       TAG       |
  | :----------: | :---------------------------------------------------------: | :-------------: |
  | 7011b5aedf78 | libhns: Add rq inline data support for hip08 RoCE user mode | rdma-core 16.0  |
  | 86188a8810ed |           RDMA/hns: Add rq inline flags judgement           | kernel 4.18-rc1 |

  openeuler/rdma 使能信息

  |   COMMITID   |                           SUBJECT                           | openeuler ENABLED(Y/N) |
  | :----------: | :---------------------------------------------------------: | :--------------------: |
  | 7011b5aedf78 | libhns: Add rq inline data support for hip08 RoCE user mode |           Y            |
  | 86188a8810ed |           RDMA/hns: Add rq inline flags judgement           |           Y            |

### 特性10: 支持超时重传配置

- **特性详解**

  支持ACK超时重传和RNR超时重传，并支持配置重传计时器用于超时重传，支持通过应用设置重传次数及重传间隔。

- **软件接口**

  | 软件接口        | 描述                                                         |
  | --------------- | ------------------------------------------------------------ |
  | ibv_modify_qp() | 通过此接口可配置重传(rnr_retry)和超时、psn_err重传次数计数器(retry_cnt) |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | 0e40dc2f70cd | [RDMA/hns: Add timer allocation support for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0e40dc2f70cd) | kernel 5.1-rc1 |
  | 441c88d5b3ff | [RDMA/hns: Fix cmdq parameter of querying pf timer resource](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=441c88d5b3ff) | kernel 5.7-rc6 |
  | 887803db866a | [RDMA/hns: Bugfix for qpc/cqc timer configuration](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=887803db866a) | kernel 5.5-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                          SUBJECT                           | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :--------------------------------------------------------: | :-----------------------------: |
  | 0e40dc2f70cd |      RDMA/hns: Add timer allocation support for hip08      |                Y                |
  | 441c88d5b3ff | RDMA/hns: Fix cmdq parameter of querying pf timer resource |                Y                |
  | 887803db866a |      RDMA/hns: Bugfix for qpc/cqc timer configuration      |                Y                |

### 特性11: 支持地址管理

#### 支持源地址管理和查询

- **特性详解**

  RoCEv1报文通信基于以太网，RoCEv2报文通信基于以太网、IP网络，但是，用户使用时不直接使用以太MAC地址和IP地址，而是基于GID进行通信。驱动将完成GID、MAC地址、IP地址之间的关联与绑定。HNS RoCE设备与HNS3网络设备共用MAC地址和IP地址。

- **软件接口**

  RoCE不提供直接的地址管理接口，复用传统网络协议栈中的地址管理机制。用户可以通过传统以太网管理工具包iproute2、net-tools中的工具进行地址管理，具体配置[参考 nic 模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)。RoCE提供以下接口进行GID的查询功能。

  | 软件接口                                                     | 描述                                                         |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | ibv_query_gid()、ibv_query_gid_ex()、ibv_query_gid_table()、rdma_query_gid() | 检索端口的全局标识符（GID）表中的条目，每个端口都被分配至少一个GID |

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | d6d91e46210f | [RDMA/hns: Add support for configuring GMV table](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b762bf9b0516) | kernel 5.11-rc1 |
  | 32053e584e4a | [RDMA/hns: Add support for filling GMV table](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32053e584e4a) | kernel 5.11-rc1 |
  | 7af80c02c7b3 | [RDMA/hns: Fix double free of the pointer to TSQ/TPQ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7af80c02c7b3) | kernel 5.11-rc1 |
  | 7406c0036f85 | [RDMA/hns: Only record vlan info for HIP08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7406c0036f85) |   kernel 5.11   |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                       SUBJECT                       | openeuler OLK-5.10 ENABLE（Y/N） |
  | :----------: | :-------------------------------------------------: | :------------------------------: |
  | b762bf9b0516 |   RDMA/hns: Add support for configuring GMV table   |                Y                 |
  | 115a32e39d13 |     RDMA/hns: Add support for filling GMV table     |                Y                 |
  | 86d6d8ac4bb3 | RDMA/hns: Fix double free of the pointer to TSQ/TPQ |                Y                 |
  | 481cd35130bc |            RDMA/hns: Only record vlan in            |                Y                 |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                        SUBJECT                         |      TAG       |
  | :----------: | :----------------------------------------------------: | :------------: |
  | 63b8f309c6af | libhns: Add verbs of querying device and querying port | rdma-core 12.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |                        SUBJECT                         | openeuler ENABLED（Y/N） |
  | :----------: | :----------------------------------------------------: | :----------------------: |
  | 63b8f309c6af | libhns: Add verbs of querying device and querying port |            Y             |

#### 支持目的地址管理

- **特性详解**

  AH全称为Address Handle，Address Handle为描述到本端的QP路径。在RDMA协议中，地址包括GID、端口号等等信息。对于RoCE而言，AH实现了MAC地址，GID，VLAN信息的统一管理。

  AH 包含到达远程目的地所需的所有数据。RC模式下，通过modify QP设置通信中使用的AH。在UD模式下，通过下发WR设置通信使用的AH。

- **软件接口**

  | 软件接口                                        | 描述                               |
  | :---------------------------------------------- | ---------------------------------- |
  | ibv_modify_qp()                                 | 支持通过modify QP参数ah_attr设置AH |
  | ibv_create_ah()                                 | 创建AH                             |
  | ibv_create_ah_from_wc()、ib_create_ah_from_wc() | 从WC中读取地址信息，创建或初始化AH |
  
- **内核态涉及代码与使能**

  参考支持源地址管理和查询。

#### 支持VLAN

- **特性详解**

  支持通过VLAN进行地址隔离。支持RoCE用户通过指定SGID的方式指定使用VLAN。

  HNS RoCE仅支持单层VLAN。

- **软件接口**

  HNS RoCE 设备的 VLAN 与HNS3网络设备共享配置。可以利用现有的网络管理工具配置，简单的使用步骤如下：

  ```bash
  #网口配置VLAN
  vconfig add eth1 1
  #配置VLAN IP
  ifconfig eth1.1 x.x.x.x
  #查看VLAN对应的GID
  ibv_devinfo -vvv | grep GID
  
  GID[  0]:               fe80:0000:0000:0000:00f4:f0ff:fe3d:29ae, RoCE v1             
  GID[  1]:               fe80::f4:f0ff:fe3d:29ae, RoCE v2
  GID[  2]:               0000:0000:0000:0000:0000:ffff:c0a8:1415, RoCE v1             
  GID[  3]:               ::ffff:192.168.20.21, RoCE v2   
  GID[  4]:               0000:0000:0000:0000:0000:ffff:0a0a:0a0a, RoCE v1             
  GID[  5]:               ::ffff:10.10.10.10, RoCE v2     
  GID[  6]:               fe80:0000:0000:0000:00f4:f0ff:fe3d:29ae, RoCE v1             
  GID[  7]:               fe80::f4:f0ff:fe3d:29ae, RoCE v2
  ```

  具体配置[参考nic模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)

- **涉及代码与使能**

  参考支持源地址管理和查询。

### 特性12: 支持事件管理

#### 支持EQ管理
- **特性详解**

  支持EQ(Event Queue) 事件队列生命周期管理，每个事件队列映射1个中断。

  EQ事件队列分为两种：异步事件队列AEQ和完成事件队列CEQ。其中AEQ用于处理硬件异常，CEQ用于处理CQE的中断上报。

  HNS RoCE支持单芯片512个EQ，支持最大8M深度，支持配置与查询EQ对应中断的亲和性。

- **软件接口**

  用户在创建CQ时可以配置所使用的CEQ，详见CQ章节。

  CEQ亲和性修改方法[参考nic 模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)。

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 247fc16d734d | [RDMA/hns: Add support for EQE in size of 64 Bytes](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=247fc16d734d) | kernel 5.10-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                      SUBJECT                      | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :-----------------------------------------------: | :-----------------------------: |
  | 247fc16d734d | RDMA/hns: Add support for EQE in size of 64 Bytes |                Y                |

#### 支持完成事件上报
- **特性详解**

  支持配置/使用RDMA 事件模式，支持完成事件上报功能。

- **软件接口**

  用户在创建CQ时可以配置使能事件模式，详见CQ章节。

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | 9a4435375cd1 | [IB/hns: Add driver files for hns RoCE driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a4435375cd1) | kernel 4.9-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                   SUBJECT                    | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :------------------------------------------: | :-----------------------------: |
  | 9a4435375cd1 | IB/hns: Add driver files for hns RoCE driver |                Y                |
#### 支持异步事件上报

- **特性详解**

  支持RDMA的异步事件上报机制，可以将软/硬件的事件上报用户。

  支持以下的事件上报功能：

  1. 支持QP事件上报

     IB_EVENT_QP_FATAL: QP上发生的错误，将QP转换为错误状态

     IB_EVENT_QP_REQ_ERR: 无效请求本地工作队列错误

     IB_EVENT_QP_ACCESS_ERR: 本地访问冲突错误

     IB_EVENT_COMM_EST: 在QP上建立通信

     IB_EVENT_SQ_DRAINED: 发送队列中正在进行的未完成的消息已被耗尽

     IB_EVENT_PATH_MIG: 连接已迁移到备用路径

     IB_EVENT_PATH_MIG_ERR：链接路径迁移错误

     IB_EVENT_QP_LAST_WQE_REACHED: 与SRQ相关联的QP上的最后一个WQE已被消耗

  2. 支持CQ事件上报

     IB_EVENT_CQ_ERR：CQ错误(CQ溢出)

  3. 支持SRQ事件上报

     IB_EVENT_SRQ_ERR：SRQ上发生的错误

     IB_EVENT_SRQ_LIMIT_REACHED：已经使用到SRQ上限

  4. 支持PORT事件上报

     IB_EVENT_PORT_ACTIVE：端口上的链路变为活动状态

     IB_EVENT_PORT_ERR：端口上的链路变为不可用状态

     注：OFED框架也提供了IBV_EVENT_GID_CHANGE等事件的上报

  5. 支持Link事件快速上报

  | 基本场景 | 预制条件                   | 输入             | 输出                                                         |
  | -------- | -------------------------- | ---------------- | :----------------------------------------------------------- |
  | UP->DOWN | 驱动加载完成 ，设备状态UP  | 触发设备DOWN     | 驱动发送IB_EVENT_PORT_ERR事件                                |
  | DOWN->UP | 驱动加载完成，设备状态DOWN | 触发设备UP       | 驱动发送IB_EVENT_PORT_ACTIVE事件                             |
  | CHANGE   | 驱动加载完成，设备状态正常 | 触发设备状态变化 | 驱动根据最新状态发送IB_EVENT_PORT_ERR或IB_EVENT_PORT_ACTIVE事件 |

- **软件接口**

  | 软件接口                                     | 接口描述                                                    |
  | -------------------------------------------- | ----------------------------------------------------------- |
  | ibv_get_async_event()、ibv_ack_async_event() | 获取异步事件返回异步事件的信息                              |
  | ib_register_event_handler()                  | 内核态用户注册事件的handler，RDMA框架以及驱动完成事件的上报 |

- **内核态涉及代码与使能**

  linux 主线合入信息

  | COMMITID     | SUBJECT                                                      | TAG            |
  | ------------ | ------------------------------------------------------------ | -------------- |
  | 9a4435375cd1 | [IB/hns: Add driver files for hns RoCE driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9a4435375cd1) | kernel 4.9-rc1 |

  openeuler OLK-5.10使能信息

  |   COMMITID   |                         SUBJECT                          | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :------------------------------------------------------: | :-----------------------------: |
  | 9a4435375cd1 |       IB/hns: Add driver files for hns RoCE driver       |                Y                |
  | c7cf81b52c02 | RDMA/hns: Add support for sending port down event fastly |                Y                |
  | 5d705e82752a |        RDMA/hns: Deliver net device event to ofed        |                Y                |

### 特性13: 支持QoS

- **特性详解**

  支持TM(Traffic Management)，支持发送方向基于Port、TC、funciton的队列管理和调度，调度算法为SP+DWRR。支持配置优先级（VLAN.pri/DSCP）到TC的映射。

  支持基于SL（server level）优先级队列的链路层端到端流控。支持8个SL，对应VLAN priority 0~7 八个优先级。

  支持基于traffic class的端到端流控，支持64个traffic class对应IPv6 DSCP traffic class中的64个class。

- **软件接口**

  具体使用方法如下：

  1. 内部优先级配置

     流量调度配置命令示例：

     ```shell
     dcb ets set dev ethX prio-tc 0:0 1:0 2:0 3:0 tc-tsa 0:ets 1:strict 2:strict 3:strict tc-bw 0:100 1:0 2:0 3:0
     ```

  2. SL (对应VLAN.pri)的配置与使用

     配置prio-tc映射示例:

     ```shell
     dcb ets set dev ethX prio-tc 0:0 1:1 2:2 3:3
     ```

     使用中，RC通过modify qp配置AH控制使用的SL，UD通过post_send关联的AH控制使用的SL。

  3. Traffic class (对应ipv6 DSCP)的配置与使用

     首先配置vlan.pri到TC的映射：

     ```
     dcb ets set dev ethX prio-tc 0:0 1:1 2:2 3:3
     ```

     配置DSCP到vlan.prio映射示例（DSCP范围0~63，priority范围0~7）：

     ```
     dcb app add dev ethX dscp-prio 1:0 4:1 8:2 16:3……
     ```

     使用中，RC通过modify qp配置AH中的GRH控制使用的traffic class，UD通过post_send关联的AH中的GRH控制使用的traffic class。

     注意事项：

     1. DSCP（tclass）仅支持RoCE v2协议；
     2. VLAN.pri(SL)与DSCP(tclass)同时使用时，调度优先级以VLAN.pri为准。报文中会同时带有VLAN.pri和DSCP；
     3. 不支持带流量更改SL与TC的映射关系，不支持带流更改DSCP与SL的映射关系。
  
- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | cacde272dd00 | [net: hns3: Add hclge_dcb module for the support of DCB feature](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=linux-5.10.y&id=cacde272dd00) | kernel 4.15-rc1 |
  | fba429fcf9a5 | [RDMA/hns: Fix missing fields in address vector](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fba429fcf9a5) |   kernel 5.11   |
  
  openeuler OLK-5.10 使能信息
  
  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | cacde272dd00 | net: hns3: Add hclge_dcb module for the support of DCB feature |                Y                |
  | 6d67845961fe |        RDMA/hns: Fix missing fields in address vector        |                Y                |
  | 11ef2ec6aa7c |             RDMA/hns: Support DSCP of userspace              |                Y                |
  
- **用户态涉及代码与使能**

  openeuler/rdma 使能信息

  |   COMMITID   |        SUBJECT        | openeuler ENABLED(Y/N) |
  | :----------: | :-------------------: | :--------------------: |
  | 12d2a17d404e | Update kernel headers |           Y            |
  | b88e6ae3e144 | libhns: Support DSCP  |           Y            |

### 特性14: 支持拥塞控制

HNS RoCE当前支持四种类型的拥塞控制算法：DCQCN、LDCP、HC3和DCQCN DIP（后文简称DIP）。HNS RoCE支持队列级别的算法类型配置以及Port级别的参数配置。

+ DCQCN：Data Center Quantized Congestion Notification，目前在RoCE v2网络中使用最广泛的拥塞控制算法。该算法融合了QCN算法和DCTCP算法，基于PFC和ECN机制进行速率控制，可以提供较好的公平性，实现高带宽利用率，保证较低的队列缓存占用率和较少的队列缓存抖动。

+ LDCP：Low Delay Control Protocol，华为罗素开发部开发的算法，是基于ACK或read response是否携带ECN标志，调整发送速率，LDCP能够实现低时延，在稳态下无拥塞丢包。

+ HC3：Huawei Converged Congestion Control，基于Host控制和主动拥塞控制，具有节点协同速率控制较好和目的端反馈快速收敛的优点。

+ DIP：Destination IP，基于目的IP的拥塞控制算法，在多QP的场景下，具有较好的效果。

#### 支持拥塞控制算法类型配置

- **特性详解**

  当前提供两套配置方案：动态配置和静态配置，同时也提供了默认算法。其中，动态配置方案为，在用户态通过`hnsdv_create_qp()`接口的hns_qp_attr参数配置不同的拥塞控制算法；静态配置方案为，通过BIOS选项选择使用的算法；在用户不进行任何算法选择时，按照固件使能优先级来选择默认算法，默认算法顺序为：DCQCN、LDCP、HC3、DIP。

- **软件接口**

  具体使用方法如下：

  1. 动态配置用户态接口：

     使用动态配置拥塞控制算法时，需要在调用`hnsdv_create_qp()`接口创建QP时，在参数hns_qp_attr结构体中，将comp_mask置上标志位HNSDV_QP_INIT_ATTR_MASK_QP_CONGEST_TYPE，然后将congest_type配置为hnsdv_qp_congest_ctrl_type枚举类型中的其中一种算法 。

     ```c
     #include <rdma/hnsdv.h>
     
     struct ibv_qp *hnsdv_create_qp(struct ibv_context *context,
     			       struct ibv_qp_init_attr_ex *qp_attr,
   			       struct hnsdv_qp_init_attr *hns_qp_attr);
     
     enum hnsdv_qp_congest_ctrl_type {
        	HNSDV_QP_CREATE_ENABLE_DCQCN = 1 << 0,
        	HNSDV_QP_CREATE_ENABLE_LDCP = 1 << 1,
        	HNSDV_QP_CREATE_ENABLE_HC3 = 1 << 2,
        	HNSDV_QP_CREATE_ENABLE_DIP = 1 << 3,
        };
     
        struct hnsdv_context {
        	uint64_t comp_mask; /* use enum hnsdv_query_context_comp_mask */
        	uint64_t flags;
        	uint8_t congest_cap; /* Use enum hnsdv_qp_congest_ctrl_type */
        };
     ```
     
  2. 静态配置
  
     在BIOS选项中选择0（DCQCN）、1（LDCP）、2（HC3）、3（DIP）来配置固定算法。 

- **内核态涉及代码与使能**

  linux 主线合入信息

  | COMMITID     | SUBJECT                                                      | TAG            |
  | ------------ | ------------------------------------------------------------ | -------------- |
  | aa84fa18741b | [RDMA/hns: Add SCC context clr support for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aa84fa18741b) | kernel 5.1-rc1 |
  | 6a157f7d1b14 | [RDMA/hns: Add SCC context allocation support for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6a157f7d1b14) | kernel 5.1-rc1 |
  | 6ac16e403900 | [RDMA/hns: Bugfix for set hem of SCC](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6ac16e403900) | kernel 5.1-rc7 |
  | 00fb67ec6b98 | [RDMA/hns: Bugfix for SCC hem free](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=00fb67ec6b98) | kernel 5.2-rc1 |
  | faa63656fc36 | [RDMA/hns: Add new command to support query vf caps](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=faa63656fc36) | kernel 6.4-rc1 |

  openeuler OLK-5.10使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | aa84fa18741b |       RDMA/hns: Add SCC context clr support for hip08        |                Y                |
  | 6a157f7d1b14 |    RDMA/hns: Add SCC context allocation support for hip08    |                Y                |
  | 6ac16e403900 |             RDMA/hns: Bugfix for set hem of SCC              |                Y                |
  | 00fb67ec6b98 |              RDMA/hns: Bugfix for SCC hem free               |                Y                |
  | 1890b7dda6ba |      RDMA/hns: Add new command to support query vf caps      |                Y                |
  | 87d0ab38d4f0 |           RDMA/hns: Modify congestion abbreviation           |                Y                |
  | 09f1b7cb29b2 | RDMA/hns: Support congestion control algorithm configuration at QP granularity |                Y                |

- **用户态涉及代码与使能**

  openeuler/rdma 使能信息

  |   COMMITID   |                          SUBJECT                           | openeuler ENABLED(Y/N) |
  | :----------: | :--------------------------------------------------------: | :--------------------: |
  | 39c7b8eaeb3a |                   Update kernel headers                    |           Y            |
  | 1b3ec79e4d61 | libhns: Support congestion control algorithm configuration |           Y            |

#### 支持拥塞控制算法模板参数配置

- **特性详解**

  支持通过RoCE的SYSFS查询或者配置四种拥塞控制算法的参数，以实现动态参数的调整。

- **软件接口**

  驱动为每种算法在” /sys/class/infiniband/$devname/ports/1/cc_param”目录下为所有的拥塞控制算法建立一个目录，分别有：dcqcn_param、ldcp_param、hc3_param、dip_param四个目录，对应四种不同的算法，这些目录下的所有文件权限都为644。

  每种算法的每一个参数都对应该算法目录下的一个文件，用户可以通过读写该文件的方式实现对参数的调整。另外，每个目录下有一个lifespan文件，该文件代表一个时间周期，在该时间周期内对参数的多次修改，将被集中下发到固件中，以避免频繁的触发固件命令的下发动作。

  DCQCN和DIP算法的参数：

  | Parameter | Description                                                  | 参数范围       |
  | :-------: | :----------------------------------------------------------- | -------------- |
  |    ai     | 设置网口eth[X]目标速率的固定步长（Additive Increate）        | [0, 65535]     |
  |     f     | 设置网口eth[X]后续迭代计数（numer of successive iterations  for fast recovery） | [0, 255]       |
  |    tkp    | 设置网口eth[X] token桶更新周期的偏移（2的N次方，如配置为8，则为2的8次方） | [0, 15]        |
  |    tmp    | 设置网口eth[X] timer更新周期的偏移                           | [0, 15]        |
  |    alp    | 设置网口eth[X] alpha更新周期                                 | [0, 65535]     |
  |     g     | 设置网口eth[X]用来更新alpha的参数G的偏移                     | [0, 15]        |
  |    al     | 设置网口eth[X] alpha的最小值                                 | [0, 255]       |
  | max_speed | 设置网口eth[X]的DCQCN目标速率的上限值（和实际发送最大速率有一定比例关系） | [0, UINT_MAX ] |
  |  ashift   | 设置网口eth[X] alpha桶更新周期的偏移（2的N次方，如配置为7，则为2的7次方） | [0, 15]        |
  | cnp_time  | 设置网口eth[X] CNP过滤周期                                   | [0, 255]       |

  LDCP算法的参数：

  | Parameter | Description                              | 参数范围       |
  | --------- | ---------------------------------------- | -------------- |
  | cwd0      | 初始窗口                                 | [0, UINT_MAX ] |
  | alpha     | 推荐1，表示CW.>1时每次增加1/cw           | [0, 255]       |
  | gamma     | 推荐0.25，表示CW<1时最小窗口、增加窗口值 | [0, 255]       |
  | beta      | 推荐0.5，表示CW>1时表示每次减少          | [0, 255]       |
  | eta       | 推荐0.5表示CW<1时每次减少0.25倍          | [0, 255]       |

  HC3算法参数：

  | Parameter        | Description                    | 参数范围       |
  | ---------------- | ------------------------------ | -------------- |
  | initial_window   | 初始窗口                       | [0, UINT_MAX ] |
  | qlen_shift       | Kmax in ReD,统一使用B为单位，  | [0, 255 ]      |
  | bandwidth        | 端口带宽，单位Mbps             | [0, UINT_MAX ] |
  | port_usage_shift | 128表示1，端口利用率，推荐0.95 | [0, 255 ]      |
  | gamma_shift      | 128表示1，推荐0.2              | [0, 15 ]       |
  | over_period      | Base RTT                       | [0, 255 ]      |
  | max_stage        | 线性加阶段次数                 | [0, 255 ]      |

  注：UINT_MAX表示整形最大数值

- **用户态涉及代码与使能**

  openeuler/rdma 使能信息

  | COMMITID |                          SUBJECT                           | openeuler ENABLED(Y/N) |
  | :------: | :--------------------------------------------------------: | :--------------------: |
  | 48ccef6  | libhns: Support congestion control algorithm configuration |           Y            |

### 特性15: 支持Bonding

- **特性详解**

  Bonding是指将多个网卡接口绑定成一个虚拟接口，将多个并行物理链路聚合成一个逻辑链路的技术。主要目的是通过一定的负载均衡策略和备份策略来提高网络的灵活性和可靠性。

  HNS RoCE Bonding支持模式1（active-backup）、模式2（XOR）、模式4（802.3ad）三种Bonding模式。

  注：使用bonding时需要搭配正确的交换机配置；例如：使用模式2时需要将对接的交换机端口配置为静态链路聚合模式。使用模式4时需要将对接的交换机端口配置为LACP模式。

  HNS RoCE Bonding与HNS3网络设备共享配置。当用户配置HNS3网络设备Bonding后，如果处于上述三种模式，且满足使用约束时，将会自动触发HNS RoCE驱动创建HNS RoCE Bonding设备。

  启用RoCE Bonding后，RoCE用户侧只能看到绑定后的虚拟设备，但是对其的操作都与物理设备保持一致，包括打开/关闭，发送/接收数据等，这个设备也有唯一的IP和MAC地址。RoCE Bonding设备命名为hns_bond_*。

  HNS RoCE Bonding的使用约束如下：

  1.不支持VF组Bond，不支持在Bond组成员上创建VF。

  2.不支持跨IO DIE组Bond

  3.不支持带业务组Bond

  4.最多支持2个RoCE Bond组，每个Bond组最多支持4个成员；仅在网口Bond组中有2个及以上的成员时才会触发RoCE Bonding；当网口Bond组中成员减少至1个以下时，RoCE Bonding自动解除

  5.目前该功能仅支持openeuler 22.03 SP1及以后版本

  6.不支持link事件快速上报

- **软件接口**

  Bonding常见命令行如下：

  ```bash
  #将eth1、eth2添加到bond0中
  ifenslave bond0 eth1 eth2
  #将eth1、eth2从bond0中删除
  ifenslave -d bond0 eth1 eth2
  #将bond0的主设备切换为eth2（仅模式1）
  ifenslave -c bond0 eth2
  #查看bond0设备信息
  cat /proc/net/bonding/bond0
  ```

  Bond设备配置方法详见[官方文档](https://www.kernel.org/doc/Documentation/networking/bonding.txt)。

- **内核态涉及代码与使能**

  openeuler OLK-5.10 使能信息

  | COMMITID     | SUBJECT                                                      | openeuler OLK-5.10 ENABLED(Y/N) |
  | ------------ | ------------------------------------------------------------ | :-----------------------------: |
  | e62a20278f18 | RDMA/hns: support RoCE bonding                               |                Y                |
  | 646b97dbb8dc | RDMA/hns: adjust the structure of RoCE bonding driver        |                Y                |
  | 6ba084e0f031 | RDMA/hns: add constraints for bonding-unsupported situations |                Y                |
  | b6623fd2c6dc | RDMA/hns: fix possible dead lock when setting RoCE Bonding   |                Y                |
  | 4920275aafcb | RDMA/hns: fix the error of missing GID in RoCE bonding mode 1 |                Y                |
  | a1598d8616e7 | RDMA/hns: fix the error of RoCE VF based on RoCE Bonding PF  |                Y                |
  | 8aeaa67170fe | RDMA/hns: Move bond_work from hns_roce_dev to hns_roce_bond_group |                Y                |
  | 82ee5d30a22d | RDMA/hns: Apply XArray for Bond ID allocation                |                Y                |
  | 41adb38e9111 | RDMA/hns: Delete a useless assignment to bond_state          |                Y                |
  | 960644465622 | RDMA/hns: Initial value assignment cleanup for RoCE Bonding variables |                Y                |
  | 9003ac2aa412 | RDMA/hns: Remove the struct member 'bond_grp' from hns_roce_dev |                Y                |
  | bc80b7288d5c | RDMA/hns: Simplify the slave uninit logic of RoCE bonding operations |                Y                |
  | 8d3ee0ae6b33 | RDMA/hns: Fix the driver uninit order during bond setting    |                Y                |
  | 01c810c8b67f | RDMA/hns: Fix the counting error of slave number             |                Y                |
  | b0f80ad22f96 | RDMA/hns: Support reset recovery for RoCE bonding            |                Y                |
  | e4ad37eabbdb | RDMA/hns: Get real-time port state of bonding slave          |                Y                |
  | c65db67aed44 | RDMA/hns: Set IB port state depending on upper device for RoCE bonding |                Y                |
  | 49f27d3e0831 | RDMA/hns: Support dispatching IB event for RoCE bonding      |                Y                |
  | a37994ad169d | RDMA/hns: Fix a missing constraint for slave num in RoCE Bonding |                Y                |
  | b5db302586ab | RDMA/hns: Rename hns_roce_bond_info_record() to make sense   |                Y                |
  | 483ccd4400f6 | RDMA/hns: Fix the repetitive workqueue mission in RoCE Bonding |                Y                |
  | ab97048c4804 | RDMA/hns: Fix the counting error of bonding with more than 2 slaves |                Y                |


### 特性16: 支持 SRIOV 虚拟化

- **特性详解**

  HNS RoCE支持SRIOV 虚拟化，单芯片支持256个function：8个PF和248个VF。支持虚拟机与容器的PF、VF硬直通。

  RoCE的VF依赖HNS3，简单的使用步骤如下，具体信息[参考nic 模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)：

  ```bash
  #加载VF驱动
  modprobe hclgevf
  #加载RoCE驱动
  modprobe hns_roce_hw_v2
  #查询最大VF规格
  cat /sys/class/net/${net_dev}/device/sriov_totalvfs
  #使能vf_num个VF
  echo ${vf_num} > /sys/class/net/${net_dev}/device/sriov_numvfs
  ```

- **软件接口**

  NA，HNS RoCE VF与PF的编程接口一致，无需特殊处理。

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |     TAG     |
  | :----------: | :----------------------------------------------------------: | :---------: |
  | 0b567cde9d7a | [RDMA/hns: Enable RoCE on virtual functions](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0b567cde9d7a) | kernel 5.13 |
  | accfc1affe9e | [RDMA/hns: Set parameters of all the functions belong to a PF](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=accfc1affe9e) | kernel 5.13 |
  | 5b03a4226c42 | [RDMA/hns: Query the number of functions supported by the PF](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5b03a4226c42) | kernel 5.13 |
  | 719d13415f59 | [RDMA/hns: Remove duplicated hem page size config code](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=719d13415f59) | kernel 5.13 |
  | 2a424e1d112a | [RDMA/hns: Reserve the resource for the VFs](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2a424e1d112a) | kernel 5.13 |
  | 0fb46da051ae | [RDMA/hns: Simplify function's resource related command](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0fb46da051ae) | kernel 5.13 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | 95d776135a46 |          RDMA/hns: Enable RoCE on virtual functions          |                Y                |
  | 66274aac3d95 | RDMA/hns: Set parameters of all the functions belong to a PF |                Y                |
  | 243bdeb15f32 | RDMA/hns: Query the number of functions supported by the PF  |                Y                |
  | 0abc8a80ce09 |    RDMA/hns: Remove duplicated hem page size config code     |                Y                |
  | 74e1531c3e24 |          RDMA/hns: Reserve the resource for the VFs          |                Y                |
  | 22d226239a08 |    RDMA/hns: Simplify function's resource related command    |                Y                |

### 特性17: 支持复位

- **特性详解**

  支持对HNS RoCE硬件错误和异常的检测与复位恢复。当硬件发生异常时，驱动将检测到异常并进行复位恢复。由于RoCE引擎和NIC引擎共PCIe function，使用相同的物理网口，所以无论是RoCEE还是NIC的硬件错误都将联动两者同时复位进行恢复。

  HNS RoCE驱动支持三种级别的复位：global复位、固件复位、function复位，前两者复位影响该芯片上所有的RoCE function，最后一个仅影响本function。

  注：触发复位后将导致用户申请的资源注销从而导致业务中断，发生复位时驱动将上报异常事件通知用户。

- **软件接口**

  [参考nic 模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)

  用户可以通过以下方式主动触发复位：

  ```shell
  #pf复位
  ethtool --reset eth1 dedicated
  #global复位
  ethtool --reset eth1 all
  #固件复位
  ethtool --reset eth1 mgmt
  #FLR复位：
  echo 1 > /sys/bus/pci/devices/0000:39:00.0/reset
  ```

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | 89a6da3cb8f3 | [RDMA/hns: reset function when removing module](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=89a6da3cb8f3) | kernel 5.3-rc1 |
  | 0fb46da051ae | [RDMA/hns: Simplify function's resource related command](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0fb46da051ae) |  kernel 5.13   |
  | 5b03a4226c42 | [RDMA/hns: Query the number of functions supported by the PF](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5b03a4226c42) |  kernel 5.13   |
  | 2a424e1d112a | [RDMA/hns: Reserve the resource for the VFs](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2a424e1d112a) |  kernel 5.13   |
  | accfc1affe9e | [RDMA/hns: Set parameters of all the functions belong to a PF](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=accfc1affe9e) |  kernel 5.13   |
  | 0b567cde9d7a | [RDMA/hns: Enable RoCE on virtual functions](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0b567cde9d7a) |  kernel 5.13   |
  | 719d13415f59 | [RDMA/hns: Remove duplicated hem page size config code](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=719d13415f59) |  kernel 5.13   |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | 89a6da3cb8f3 |        RDMA/hns: reset function when removing module         |                Y                |
  | 22d226239a08 |    RDMA/hns: Simplify function's resource related command    |                Y                |
  | 243bdeb15f32 | RDMA/hns: Query the number of functions supported by the PF  |                Y                |
  | 74e1531c3e24 |          RDMA/hns: Reserve the resource for the VFs          |                Y                |
  | 66274aac3d95 | RDMA/hns: Set parameters of all the functions belong to a PF |                Y                |
  | 95d776135a46 |          RDMA/hns: Enable RoCE on virtual functions          |                Y                |
  | 0abc8a80ce09 |    RDMA/hns: Remove duplicated hem page size config code     |                Y                |

### 特性18: 支持RAS

- **特性详解**

  RAS(Reliability, Availabilty, Serviceabiliy)是指一种提高系统可靠性，可用性，可服务性的技术。简单来说，就是一套支持故障检测、上报和恢复的软硬件机制，保障RoCEE硬件在遇到故障时能够继续正确稳定地运行，并且给用户提供相关的信息用于定位故障原因。

  HNS RoCE支持RAS功能，支持RAS异常上报与恢复。

- **软件接口**

  硬件检测到一些硬件错误时，如果该故障的上报方式配置为RAS通路，则将触发RAS故障上报。具体会携带可分析的寄存器信息，可以参考[ras故障分析](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86/WayCa-Kunpeng-%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86-RAS%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)手册进行分析。

- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |       TAG       |
  | :----------: | :----------------------------------------------------------: | :-------------: |
  | 626903e9355b | [RDMA/hns: Add support for reporting wc as software mode](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=626903e9355b) | kernel 5.5-rc6  |
  | d3743fa94ccd | [RDMA/hns: Fix the chip hanging caused by sending doorbell during reset](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d3743fa94ccd) | kernel 5.1-rc1  |
  | 6a04aed6afae | [RDMA/hns: Fix the chip hanging caused by sending mailbox&CMQ during reset](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6a04aed6afae) | kernel 5.1-rc1  |
  | d061effc36f7 | [RDMA/hns: Fix the Oops during rmmod or insmod ko when reset occurs](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d061effc36f7) | kernel 5.1-rc1  |
  | 2b9acb9a97fe | [RDMA/hns: Add the process of AEQ overflow for hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2b9acb9a97fe) | kernel 5.1-rc1  |
  | cb7a94c9c808 | [RDMA/hns: Add reset process for RoCE in hip08](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cb7a94c9c808) | kernel 4.18-rc1 |
  | 3ec5f54f7a0f | [RDMA/hns: Fix an cmd queue issue when resetting](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3ec5f54f7a0f) | kernel 5.8-rc1  |
  | e075da5e7c47 | [RDMA/hns: Add reset process for function-clear](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e075da5e7c47) | kernel 5.4-rc1  |
  | 726be12f5ca0 | [RDMA/hns: Set reset flag when hw resetting](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=726be12f5ca0) | kernel 5.3-rc1  |
  | 2de949abd6a5 | [RDMA/hns: Recover 1bit-ECC error of RAM on chip](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2de949abd6a5) | kernel 6.0-rc1  |
  | e8ea058edc2b | [RDMA/hns: Add the detection for CMDQ status in the device initialization process](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e8ea058edc2b) |   kernel 5.18   |
  
  openeuler OLK-5.10 使能信息
  
  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | 626903e9355b |   RDMA/hns: Add support for reporting wc as software mode    |                Y                |
  | d3743fa94ccd | RDMA/hns: Fix the chip hanging caused by sending doorbell during reset |                Y                |
  | 6a04aed6afae | RDMA/hns: Fix the chip hanging caused by sending mailbox&CMQ during reset |                Y                |
  | d061effc36f7 | RDMA/hns: Fix the Oops during rmmod or insmod ko when reset occurs |                Y                |
  | 2b9acb9a97fe |     RDMA/hns: Add the process of AEQ overflow for hip08      |                Y                |
  | cb7a94c9c808 |        RDMA/hns: Add reset process for RoCE in hip08         |                Y                |
  | 3ec5f54f7a0f |       RDMA/hns: Fix an cmd queue issue when resetting        |                Y                |
  | e075da5e7c47 |        RDMA/hns: Add reset process for function-clear        |                Y                |
  | 726be12f5ca0 |          RDMA/hns: Set reset flag when hw resetting          |                Y                |
  | 1a8c6fa4adfb |       RDMA/hns: Recover 1bit-ECC error of RAM on chip        |                Y                |
  | 34e27e3f9408 | RDMA/hns: Add the detection for CMDQ status in the device initialization process |                Y                |


### 特性19: 支持IOPMU

- **特性详解**

  鲲鹏920板载网卡提供了IOPMU（Performance Monitoring Unit）硬件模块来增强硬件性能的可观测性，当前已将该能集成到了linux perf框架中。加载IOPMU驱动后，用户可以通过使用perf来实现对在实际业务场景下的硬件性能的观测。

- **软件接口**

  详细使用方法[参考nic模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)

  常见使用命令如下：

  ```bash
  #加载IOPMU驱动
  modprobe hns3_pmu.ko
  #查看每个事件的event和subevent
  cat /sys/devices/hisi_hns{X}/event/
  #查看每个事件支持的过滤模式
  cat /sys/devices/hisi_hns{X}/filtermode/
  #查看每个输入参数支持的位宽
  cat sys/devices/hisi_hns{X}/format/
  #支持port级别的事件统计
  perf stat -a -e hisi_hns0/port=0x0,tc=0xF,event=0xXX,subevent=0xXX/ -I 1000
  #支持port指定tc的事件统计
  perf stat -a -e hisi_hns0/port=0x0,tc=0xX,event=0xXX,subevent=0xXX/ -I 1000
  #支持function级别的事件统计
  perf stat -a -e hisi_hns0/func=0x3500, queue=0xFFFF,event=0xXX,subevent=0xXX/ -I 1000
  #支持global类型的事件统计
  perf stat -a -e hisi_hns0/global=1,event=0xXX,subevent=0xXX/ -I 1000
  ```

- **内核态涉及代码与使能**

  [参考nic模块](https://gitee.com/openeuler/WayCa/blob/master/WayCa-Kunpeng-%E5%BA%95%E8%BD%AF/Wayca-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C/WayCa-Kunpeng-%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C-%E6%9D%BF%E8%BD%BD%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md)

### 特性20: 支持RoH业务

- **特性详解**

  RoH即RDMA over HCCS。与RoCE相比，RoH的链路层不再是以太网，而替换为HCCS。使用上，RoH场景下，上层业务的数据通道不感知数据链路层的改变，RoH网络上使能RoCEv2协议业务(不支持RoCE v1)。RoH MAC模式下，网络层不支持IPv6，应用软件需要使用IPv4对应的GID。

- **软件接口**

  NA，复用RoCE接口。

- **内核态涉及代码与使能**

  详见RoH模块说明


### 特性21: 支持rdmatool

- **特性详解**

  rdmatool是网络工具套件iproute2的一部分（所在路径iproute2/rdma/rdma），它为RDMA设备提供了资源、统计、链路状态等查询功能，为用户提供了运行状态监控和故障定位的能力。

  当前HNS RoCE支持rdmatool的全部查询功能。

  rdmatool的详细说明，请查询[官方网站](https://man7.org/linux/man-pages/man8/rdma.8.html)。

- **软件接口**

  使用方法如下：

  ```shell
  #查询rdma设备信息
  rdma dev show
  #查询当前资源使用的整体情况
  rdma res show [DEV]
  #查询当前某类资源的关键信息
  rdma res show [qp|cm_id|pd|mr|cq|ctx|srq]
  #查询某类资源的context信息（信息以raw格式呈现）
  rdma res show [qp|cm_id|pd|mr|cq|ctx|srq] -r -j
  #查询rdma 设备link状态
  rdma link show [DEV/PORT_INDEX]
  #查询软/硬件统计信息
  rdma stat show link [ DEV/PORT_INDEX ] -p
  ```
  
  注意：资源的context信息以及软硬件统计信息目前仅支持openeuler 22.03 SP2版本。
  
- **内核态涉及代码与使能**

  linux 主线合入信息

  |   COMMITID   |                           SUBJECT                            |      TAG       |
  | :----------: | :----------------------------------------------------------: | :------------: |
  | e1c9a0dc2939 | [RDMA/hns: Dump detailed driver-specific CQ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e1c9a0dc2939) | kernel 5.1-rc7 |
  | 40b4b79c866f | [RDMA/hns: Remove redundant DFX file and DFX ops structure](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=40b4b79c866f) | kernel 6.1-rc1 |
  | eb00b9a08b9d | [RDMA/hns: Add or remove CQ's restrack attributes](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eb00b9a08b9d) | kernel 6.1-rc1 |
  | f2b070f36d1b | [RDMA/hns: Support CQ's restrack raw ops for hns driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f2b070f36d1b) | kernel 6.1-rc1 |
  | e198d65d76e9 | [RDMA/hns: Support QP's restrack ops for hns driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e198d65d76e9) | kernel 6.1-rc1 |
  | 3e89d78b21a8 | [RDMA/hns: Support QP's restrack raw ops for hns driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3e89d78b21a8) | kernel 6.1-rc1 |
  | dc9981ef17c6 | [RDMA/hns: Support MR's restrack ops for hns driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dc9981ef17c6) | kernel 6.1-rc1 |
  | 3d67e7e236ad | [RDMA/hns: Support MR's restrack raw ops for hns driver](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3d67e7e236ad) | kernel 6.1-rc1 |

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                          SUBJECT                          | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :-------------------------------------------------------: | :-----------------------------: |
  | e1c9a0dc2939 |        RDMA/hns: Dump detailed driver-specific CQ         |                Y                |
  | 9c9edf689a60 | RDMA/hns: Remove redundant DFX file and DFX ops structure |                Y                |
  | 9dd913cd565c |     RDMA/hns: Add or remove CQ's restrack attributes      |                Y                |
  | 78163ff31523 |  RDMA/hns: Support CQ's restrack raw ops for hns driver   |                Y                |
  | 78425b64c781 |    RDMA/hns: Support QP's restrack ops for hns driver     |                Y                |
  | 0480d2ff2ed6 |  RDMA/hns: Support QP's restrack raw ops for hns driver   |                Y                |
  | 98b07261a4da |    RDMA/hns: Support MR's restrack ops for hns driver     |                Y                |
  | 4d20406bc210 |  RDMA/hns: Support MR's restrack raw ops for hns driver   |                Y                |
  | d5a4ca75e7ca |                RDMA/hns: Add dfx cnt stats                |                Y                |
  | 05491dda8e9b |              RDMA/hns: Support hns HW stats               |                Y                |


### 特性22: 支持 DCA

- **特性详解**

  DCA特性是通过将QP的WQE内存进行进程级别共享的方式，来减少有链接通信中的内存消耗。

  使能DCA时，同一个进程的多个QP可以共享一个WQE buffer内存池。在业务下发时，这些共享内存的QP将从内存池中动态申请WQE buffer；当业务处理完毕后释放资源返回WQE buffer内存池，从而实现内存的复用，提高内存的使用率。

- **软件接口**

  **用户态：**

  用户态DCA通过Direct verbs使能。

  1. 通过DV接口打开HNS RoCE设备并配置DCA参数

     ```c
     struct ibv_context *hnsdv_open_device(struct ibv_device *device, struct hnsdv_context_attr *attr);
     ```

     使用DCA时，attr中的flag需要置位HNSDV_CONTEXT_FLAGS_DCA:

     ```c
     attr->flag |= HNSDV_CONTEXT_FLAGS_DCA;
     ```

     DCA一共有四个参数`dca_prime_qps`，`dca_unit_size`，`dca_max_size`，`dca_min_size`，每个参数对应一个comp_mask，只有comp_mask置位时，对应的参数才会生效：

     ```c
     attr->comp_mask |= HNSDV_CONTEXT_MASK_DCA_PRIME_QPS;
     attr->dca_prime_qps = 100;
     ```

     DCA 的参数信息见下表：

     | Parameter     | Description                                                  | 参数范围                            |
     | ------------- | ------------------------------------------------------------ | ----------------------------------- |
     | dca_prime_qps | 指定使用DCA的QP数量，当使用DCA的QP数量超过该值后新建的DCA QP的性能将急剧下降。 | [0, UINT_MAX ]  默认为0             |
     | dca_unit_size | WQE buffer共享池增长的步长。                                 | [0, UINT_MAX]  默认为4K             |
     | dca_max_size  | WQE buffer共享池的最大规格。                                 | [0, UINT_MAX]  默认为0，表示无限大  |
     | dca_min_size  | WQE buffer共享池的最小值，当共享池超过该阈值时，如果业务空闲将触发回收机制。 | [0, UINT_MAX]  默认等于dca_max_size |

     具体DV接口中open device相关的参数的定义如下：

     ```c
     enum hnsdv_context_attr_flags {
     	HNSDV_CONTEXT_FLAGS_DCA = 1 << 0,
     };
     enum hnsdv_context_comp_mask {
     	HNSDV_CONTEXT_MASK_DCA_PRIME_QPS = 1 << 0,
     	HNSDV_CONTEXT_MASK_DCA_UNIT_SIZE = 1 << 1,
     	HNSDV_CONTEXT_MASK_DCA_MAX_SIZE = 1 << 2,
     	HNSDV_CONTEXT_MASK_DCA_MIN_SIZE = 1 << 3,
     };
     struct hnsdv_context_attr {
     	uint64_t flags; /* Use enum hnsdv_context_attr_flags */
     	uint64_t comp_mask; /* Use enum hnsdv_context_comp_mask */
     	uint32_t dca_prime_qps;
         uint32_t dca_unit_size;
         uint64_t dca_max_size;
         uint64_t dca_min_size;
     };
     
     ```

  2. 通过DV接口创建QP，使能需要使用DCA的QP

     Context使能DCA后还需要指定需要用DCA的QP，未设置QP启用DCA时，相关的QP不会使能DCA功能。

     ```c
     struct ibv_qp *hnsdv_create_qp(struct ibv_context *context,
     
     struct ibv_qp_init_attr_ex *qp_attr, struct hnsdv_qp_init_attr *hns_qp_attr)
     ```

     使用DCA的QP需要在hns_qp_attr中将DCA的create flag置位：

     /* comp_mask置位create flag，使create flag参数生效*/

     ```c
     hns_qp_attr->comp_mask |= HNSDV_QP_INIT_ATTR_MASK_QP_CREATE_FLAGS;
     
     hns_qp_attr->create_flags |= HNSDV_QP_CREATE_ENABLE_DCA_MODE;
     ```

     Create QP相关的DV接口参数的定义如下：

     ```c
     enum hnsdv_qp_create_flags {
     	HNSDV_QP_CREATE_ENABLE_DCA_MODE = 1 << 0,
     };
      
     enum hnsdv_qp_init_attr_mask {
     	HNSDV_QP_INIT_ATTR_MASK_QP_CREATE_FLAGS = 1 << 0,
     };
     
     struct hnsdv_qp_init_attr {
     	uint64_t comp_mask;   /* Use enum hnsdv_qp_init_attr_mask */
     	uint32_t create_flags; /* Use enum hnsdv_qp_create_flags */
     };
     ```

  **内核态：**

    1. 内核态通过模块参数方式使能DCA，通过dca_max_size、dca_min_size、dca_unit_size三个参数控制DCA的使能以及内存池的范围。参数的含义同用户态。例：

       ```c
        insmod hns-roce-hw-v2.ko dca_unit_size=4096 dca_min_size=409600 dca_max_size=10485760
       ```

    2. 在创建QP时需要通过`create qp`的`ib_qp_init_attr`参数的`create_flags`域段设置`IB_QP_CREATE_RESERVED_START`标志来打开对应QP的DCA功能（默认关闭），例如：

       ```c
        init_attr->create_flags |= IB_QP_CREATE_RESERVED_START;
       ```

- **DCA的调试手段**

  为了更好的支持DCA的使用，HNS RoCE支持通过debug FS查询当前DCA的使用情况。HNS RoCE驱动在” /sys/kernel/debug/hns_roce/${dev_name}/dca/”目录下为每个使用DCA的进程建立了一个子目录，内核进程的目录名为“kernel”，其他进程则以PID为名，每个目录下存在mstate以及qp两个文件。其中，qp文件中记录了该进程下所有QP的内存使用情况，mstate记录了所有的内存使用情况，以及其他DCA的运行信息。除此之外，在dca目录下，还有pool、qp两个debugfs文件，分别用来记录所有内存池的内存使用状态，以及所有DCA QP的内存使用情况和状态。

  注意事项：DCA仅支持RC、XRC，不支持UD通信。

- **内核态涉及代码与使能**

  openeuler OLK-5.10 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :-----------------------------: |
  | 21a0d4fe7b81 | RDMA/hns: Fixes concurrent ressetting and post_recv in DCA mode |                Y                |
  | d3caaebdbfe9 | RDMA/hns: Optimize user DCA perfermance by sharing DCA status |                Y                |
  | d3caaebdbfe9 |            RDMA/hns: Add debugfs support for DCA             |                Y                |
  | d3caaebdbfe9 |          RDMA/hns: Add DCA support for kernel space          |                Y                |
  | f0384ddcf1ee |      RDMA/hns: Add method to query WQE buffer's address      |                Y                |
  | 0273952c5e6e |          RDMA/hns: Add method to detach WQE buffer           |                Y                |
  | 0cf17392d266 |  RDMA/hns: Setup the configuration of WQE addressing to QPC  |                Y                |
  | d8cca476a8d2 |        RDMA/hns: Add method for attaching WQE buffer         |                Y                |
  | 40e4b148b5bd |      RDMA/hns: Configure DCA mode for the userspace QP       |                Y                |
  | bca9ff271249 |      RDMA/hns: Add method for shrinking DCA memory pool      |                Y                |
  | f44a2f97d82a |              RDMA/hns: Introduce DCA for RC QP               |                Y                |

- **用户态涉及代码与使能**

  openeuler/rdma 使能信息

  |   COMMITID   |       SUBJECT        | openeuler OLK-5.10 ENABLED(Y/N) |
  | :----------: | :------------------: | :-----------------------------: |
  | b88a370b79cd | Support hns roce DCA |                Y                |

### 特性23: 支持TD无锁

- **特性详解**

  支持用户态TD（Thread Domain）无锁功能，这里的无锁是指在IO过程中驱动不再通过锁来保证多线程之间的资源互斥，由用户自行保证业务逻辑上不存在资源竞争。

  TD无锁实现了队列级别的无锁，支持每个CQ，SRQ，QP独立配置。 

- **软件接口**

  | 软件接口                       | 描述                              |
  | ------------------------------ | --------------------------------- |
  | ibv_alloc_td()、ibv_alloc_pd() | 创建TD、PD                        |
  | ibv_alloc_parent_domain()      | 将TD与PD关联到PAD                 |
  | ibv_create_cq_ex()             | 通过PAD创建新的资源，以QP、CQ为例 |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                           SUBJECT                            |     TAG      |
  | :----------: | :----------------------------------------------------------: | :----------: |
  | 812372fadc96 | libhns: Add support for the thread domain and the parent domain | rdma-core 28 |

  openeuler/rdma 使能信息

  |   COMMITID   |                           SUBJECT                            | openeuler  ENABLED(Y/N) |
  | :----------: | :----------------------------------------------------------: | :---------------------: |
  | 812372fadc96 | libhns: Add support for the thread domain and the parent domain |            Y            |


### 特性24: 支持OFED演进的IO接口

- **特性详解**

  支持ibv_wr和ibv_wc等OFED新增IO接口。

  场景约束：用户使用新增的QP EX的接口，用户必须严格按照start、wr、set sge和complete四个阶段的调用顺序。

- **软件接口**

  | **软件接口**                          | **描述**                                                     |
  | ------------------------------------- | ------------------------------------------------------------ |
  | ibv_create_qp_ex()                    | 创建QP                                                       |
  | ibv_wr_send()                         | 发送消息，初始化WQE, 基于IBV_WR_SEND和wr_flags(enum ibv_send_flags)配置WQE |
  | ibv_wr_rdma_write()                   | 初始化WQE，基于IBV_WR_RDMA_WRITE和wr_flags配置WQE            |
  | ibv_wr_set_inline_data()              | 配置单个Inline data，不能单独使用，需要配合opcode接口函数业务使用 |
  | ibv_wr_start()                        | 向QP发送工作请求，获取doorbell，释放SQ锁资源，初始化扩展sge信息，备份sq wqe的初始位置等 |
  | ibv_wr_complete()                     | 未检测到err，触发doorbell，释放SQ锁资源，检测到err，回滚sq wqe指针，释放锁资源 |
  | ibv_start_poll()                      | 获取cq锁资源，进行第一次poll，将数据存入hns_roce_cq中的cqe结构 |
  | ibv_wc_opcode()、ibv_wc_read_opcode() | 获取cqe中opcode字段                                          |

- **用户态涉及代码与使能**

  linux/rdma 合入信息

  |   COMMITID   |                          SUBJECT                          |       TAG       |
  | :----------: | :-------------------------------------------------------: | :-------------: |
  | 13c10af17443 |             libhns: Support ibv_create_qp_ex              | rdma-core v35.0 |
  | 36446a56eea5 | libhns: Extended QP supports the new post send mechanism  | rdma-core v40.0 |
  | d8596eff4eb4 |       libhns: Add support for creating extended CQ        | rdma-core v40.0 |
  | 0464e0cb0416 |  libhns: Extended CQ supports the new polling mechanism   | rdma-core v40.0 |
  | 2d48954e9b26 |        libhns: Optimize the error handling of CQE         | rdma-core v40.0 |
  | 9dd7b55957cc | libhns: Refactor hns roce v2 poll one() and wc poll cqe() | rdma-core v40.0 |
  | d8596eff4eb4 |       libhns: Add support for creating extended CQ        | rdma-core v40.0 |

  openeuler/rdma 使能信息

  |   COMMITID   |                          SUBJECT                          | openeuler ENABLED(Y/N) |
  | :----------: | :-------------------------------------------------------: | :--------------------: |
  | 13c10af17443 |             libhns: Support ibv_create_qp_ex              |           Y            |
  | 36446a56eea5 | libhns: Extended QP supports the new post send mechanism  |           Y            |
  | d8596eff4eb4 |       libhns: Add support for creating extended CQ        |           Y            |
  | 0464e0cb0416 |  libhns: Extended CQ supports the new polling mechanism   |           Y            |
  | 2d48954e9b26 |        libhns: Optimize the error handling of CQE         |           Y            |
  | 9dd7b55957cc | libhns: Refactor hns roce v2 poll one() and wc poll cqe() |           Y            |
  | d8596eff4eb4 |       libhns: Add support for creating extended CQ        |           Y            |



​	
