[TOC]

# openEuler WayCa SIG 鲲鹏高速网络 RoCE用户使用指南

## 软件版本

- openEuler 20.03 sp3
- openEuler 22.03 lts
- openEuler 22.03 lts sp1
- openEuler 22.03 lts sp2

## 安装使用

### 加载内核态驱动
```bash
[root@localhost ~]# modprobe hns3
[root@localhost ~]# modprobe hclge
[root@localhost ~]# modprobe hclgevf
[root@localhost ~]# modprobe hns_roce_hw_v2
```
### 安装用户态包

```bash
[root@localhost ~]# yum install rdma-core
```
### 查看设备信息
```bash
[root@localhost ~]# rdma link
link roceo1/1 state ACTIVE physical_state LINK_UP netdev eno1
link roceo2/1 state ACTIVE physical_state LINK_UP netdev eno2
link roceo3/1 state ACTIVE physical_state LINK_UP netdev eno3

[root@localhost ~]# ibv_devinfo -d roceo1
hca_id: roceo1
        transport:                      InfiniBand (0)
        fw_ver:                         0.778.2068
        node_guid:                      0cb5:efff:fe58:d5c7
        sys_image_guid:                 0cb5:efff:fe58:d5c7
        vendor_id:                      0x19e5
        vendor_part_id:                 41512
        hw_ver:                         0x130
        phys_port_cnt:                  1
                port:   1
                        state:                  PORT_ACTIVE (4)
                        max_mtu:                4096 (5)
                        active_mtu:             1024 (3)
                        sm_lid:                 0
                        port_lid:               0
                        port_lmc:               0x00
                        link_layer:             Ethernet

```


## RoCE基本功能测试

HNS RoCE依赖HNS3网络设备，因此在使用HNS RoCE前，需要保证HNS3网络设备的功能正常，基于自身组网条件，完成相应HNS3网络设备的网络配置。

在完成了基本的网络配置后，可以利用perftest工具包完成HNS RoCE的基本功能及性能测试，该测试工具包的安装方法如下：

```bash
[root@localhost ~]# yum install perftest					
```

perftest工具包括RDMA设备的多种服务类型多种模式的测试，HNS RoCE主要支持8种测试工具：ib_atomic_bw 、ib_read_bw 、ib_send_bw、ib_write_bw、ib_atomic_lat、ib_read_lat、ib_send_lat、ib_write_lat；这些测试工具分别用来测试不同的操作类型的时延和带宽性能。具体说明可以通过-h查看帮助讯息：

```bash
[root@localhost ~]#  ib_read_bw -h
```

HNS RoCE支持3种服务类型：RC、UD和XRC, 四种基础IO操作类型：send、write、read和atomic，具体信息可参考模块特性介绍部分。

### RC服务基本测试

#### send操作

+ 测试方法

```bash
[root@localhost ~]# ib_send_bw -d roceo1 -q 1 -c RC &
[root@localhost ~]# ib_send_bw -d roceo1 -q 1 -c RC 192.168.66.199 &
```
+ 测试结果

```shell
[root@localhost ~]# 
************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 ibv_wr* API     : ON
 RX depth        : 512
 TX depth        : 128
 CQ Moderation   : 1
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Mtu             : 1024[B]
 Link type       : Ethernet
 Link type       : Ethernet
 GID index       : 3
 GID index       : 3
 Max inline data : 0[B]
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x000b PSN 0xd07df
 local address: LID 0000 QPN 0x000a PSN 0xe4af32
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 remote address: LID 0000 QPN 0x000a PSN 0xe4af32
 remote address: LID 0000 QPN 0x000b PSN 0xd07df
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      1000             0.00               22724.01                   0.363584
---------------------------------------------------------------------------------------
 65536      1000             22727.48            22651.04                  0.362417
---------------------------------------------------------------------------------------
```
#### read操作
+ 测试方法

```bash
[root@localhost ~]# ib_read_bw -d roceo1  -q 1  &
[root@localhost ~]# ib_read_bw -d roceo1  -q 1 192.168.66.199 &
```
+ 测试结果

```shell
[root@localhost ~]#
************************************
* Waiting for client to connect... *
************************************
[root@localhost ~]# 
---------------------------------------------------------------------------------------
                    RDMA_Read BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
---------------------------------------------------------------------------------------
                    RDMA_Read BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Outstand reads  : 128
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 ibv_wr* API     : ON
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Outstand reads  : 128
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0022 PSN 0x1ec000 OUT 0x80 RKey 0x000300 VAddr 0x00ffffb2eb6000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 local address: LID 0000 QPN 0x0023 PSN 0x8f8272 OUT 0x80 RKey 0x000400 VAddr 0x00ffff80044000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 remote address: LID 0000 QPN 0x0023 PSN 0x8f8272 OUT 0x80 RKey 0x000400 VAddr 0x00ffff80044000
 remote address: LID 0000 QPN 0x0022 PSN 0x1ec000 OUT 0x80 RKey 0x000300 VAddr 0x00ffffb2eb6000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      1000             14880.99            14855.98                  0.237696
---------------------------------------------------------------------------------------
 65536      1000             14880.99            14855.98                  0.237696
---------------------------------------------------------------------------------------
```

#### write操作
+ 测试方法

```bash
[root@localhost ~]# ib_write_bw -d roceo1 -q 1 &
[root@localhost ~]# ib_write_bw -d roceo1 -q 1  192.168.66.199 &
```
+ 测试结果

```shell
[root@localhost ~]#
************************************
* Waiting for client to connect... *
************************************
[root@localhost ~]# 
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : roceo1
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 PCIe relax order: ON
 ibv_wr* API     : ON
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 ibv_wr* API     : ON
 Data ex. method : Ethernet
 TX depth        : 128
---------------------------------------------------------------------------------------
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x002a PSN 0x718f11 RKey 0x000300 VAddr 0x00ffffabeec000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 local address: LID 0000 QPN 0x002b PSN 0x2351c5 RKey 0x000400 VAddr 0x00ffffa9c6d000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 remote address: LID 0000 QPN 0x002b PSN 0x2351c5 RKey 0x000400 VAddr 0x00ffffa9c6d000
 remote address: LID 0000 QPN 0x002a PSN 0x718f11 RKey 0x000300 VAddr 0x00ffffabeec000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5000             22727.40            22704.19                  0.363267
---------------------------------------------------------------------------------------
 65536      5000             22727.40            22704.19                  0.363267
---------------------------------------------------------------------------------------
```

#### atomic操作
+ 测试方法

```bash
[root@localhost ~]# ib_atomic_bw -d roceo1  -q 1 &
[root@localhost ~]# ib_atomic_bw -d roceo1  -q 1 192.168.66.199 &
```
+ 测试结果

```shell
[root@localhost ~]#
************************************
* Waiting for client to connect... *
************************************
[root@localhost ~]#
---------------------------------------------------------------------------------------
                    Atomic FETCH_AND_ADD BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
---------------------------------------------------------------------------------------
                    Atomic FETCH_AND_ADD BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Outstand reads  : 128
 rdma_cm QPs     : OFF
 ibv_wr* API     : ON
 Data ex. method : Ethernet
 TX depth        : 128
---------------------------------------------------------------------------------------
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Outstand reads  : 128
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0033 PSN 0x13e6a2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 local address: LID 0000 QPN 0x0032 PSN 0xf39005
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 remote address: LID 0000 QPN 0x0032 PSN 0xf39005
 remote address: LID 0000 QPN 0x0033 PSN 0x13e6a2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 8          1000             76.29              72.43              9.493989
---------------------------------------------------------------------------------------
 8          1000             76.29              72.43              9.493989
---------------------------------------------------------------------------------------
```

### UD服务基本测试
+ 测试方法

```bash
[root@localhost ~]# ib_send_bw -d roceo1 -q 1 -c UD &
[root@localhost ~]# ib_send_bw -d roceo1 -q 1 -c UD 192.168.66.199  &
```
+ 测试结果

```bash
[root@localhost ~]#
************************************
* Waiting for client to connect... *
************************************
[root@localhost ~]#  
 Max msg size in UD is MTU 1024
 Max msg size in UD is MTU 1024
 Changing to this MTU
 Changing to this MTU
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : UD           Using SRQ      : OFF
 PCIe relax order: ON
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : roceo1
 Number of qps   : 1            Transport type : IB
 Connection type : UD           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 ibv_wr* API     : ON
 TX depth        : 128
 RX depth        : 1000
 CQ Moderation   : 100
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Mtu             : 1024[B]
 Link type       : Ethernet
 Link type       : Ethernet
 GID index       : 3
 GID index       : 3
 Max inline data : 0[B]
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0002 PSN 0x46127e
 local address: LID 0000 QPN 0x0003 PSN 0x385faa
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 remote address: LID 0000 QPN 0x0003 PSN 0x385faa
 remote address: LID 0000 QPN 0x0002 PSN 0x46127e
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 1024       1000             0.00               7876.21            8.065242 1024       1000             8138.01            7630.57                 7.813706

---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
```

### XRC服务基本测试
+ 测试方法

```bash
[root@localhost ~]# ib_write_bw -d roceo1 -q 1 -c XRC &
[root@localhost ~]# ib_write_bw -d roceo1 -q 1 -c XRC 192.168.66.199 &
```
+ 测试结果

```shell
[root@localhost ~]#
************************************
* Waiting for client to connect... *
************************************
[root@localhost ~]# 
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
---------------------------------------------------------------------------------------
 Dual-port       : OFF          Device         : roceo1
                    RDMA_Write BW Test
 Number of qps   : 1            Transport type : IB
 Dual-port       : OFF          Device         : roceo1
 Connection type : XRC          Using SRQ      : ON
 Number of qps   : 1            Transport type : IB
 PCIe relax order: ON
 Connection type : XRC          Using SRQ      : ON
 PCIe relax order: ON
 ibv_wr* API     : ON
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 1
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Mtu             : 1024[B]
 Link type       : Ethernet
 Link type       : Ethernet
 GID index       : 3
 GID index       : 3
 Max inline data : 0[B]
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0012 PSN 0xae93b6 RKey 0x000300 VAddr 0x00ffffb1072000 SRQn 00000000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 local address: LID 0000 QPN 0x0013 PSN 0x1b5520 RKey 0x000400 VAddr 0x00ffffa1730000 SRQn 0x000001
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 remote address: LID 0000 QPN 0x0013 PSN 0x1b5520 RKey 0x000400 VAddr 0x00ffffa1730000 SRQn 0x000001
 remote address: LID 0000 QPN 0x0012 PSN 0xae93b6 RKey 0x000300 VAddr 0x00ffffb1072000 SRQn 00000000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:66:199
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5000             22727.55            22704.79                  0.363277
---------------------------------------------------------------------------------------
 65536      5000             22727.55            22704.79                  0.363277
---------------------------------------------------------------------------------------
```