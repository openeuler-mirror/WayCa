# openEuler WayCa SIG 加速器介绍

* 加速器模块包含基于CRYPTO、HWRNG、UACCE、UADK框架实现的支持鲲鹏加速器设备的内核态与用户态驱动。加速器模块包括5个子模块，分别为队列管理模块QM、高性能RSA计算引擎HPRE、硬件安全加速引擎SEC、压缩算法加速引擎ZIP、真随机数产生器TRNG模块。QM是为了统一加速器设备和软件之间的接口，采用统一的队列管理模块QM来和软件进行交互。加速器HPRE、SEC、ZIP设备都集成QM模块。HPRE模块支持RSA/DH/ECDH/X25519/X448/ECDSA/SM2算法，ZIP模块支持gzip/zlib/deflate/lz77_zstd算法，SEC模块支持AEAD/SKCIPHER/DIGEST算法，TRNG模块支持获取随机数。加速器驱动代码已上传到linux和linaro社区。 

## ACC 模块介绍

* ACC模块是加速器内核态硬件驱动，内核态驱动主要实现加速器设备虚拟化（SRIOV）、限流、DFX信息查询、crypto内核态硬件算法、虚拟化热迁移、RAS硬件错误处理、FLR复位、低功耗等这几大特性。

### 一、内核态驱动使能

#### 源码获取路径

* linux kernel 仓库：<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git>
* openeuler 仓库：<https://gitee.com/openeuler/kernel.git> 
* 源码目录：drivers/crypto/hisilicon/

#### 内核配置

* 加速器模块，内核态驱动涉及的config选项有：

  * CONFIG_CRYPTO_DEV_HISI_QM=m			hisi_qm.ko编译配置选项
  	 CONFIG_CRYPTO_DEV_HISI_ZIP=m			hisi_zip.ko编译配置选项
  	 CONFIG_CRYPTO_DEV_HISI_HPRE=m		hisi_hpre.ko编译配置选项
  	 CONFIG_CRYPTO_DEV_HISI_SEC2=m		hisi_sec2.ko编译配置选项
  	 CONFIG_CRYPTO_DEV_HISI_TRNG=m		hisi-trng-v2.ko编译配置选项
  	 CONFIG_CRYPTO_DEV_HISI_MIGRATION=m	hisi_migration.ko编译配置选项
  	 CONFIG_UACCE=m						uacce.ko编译配置选项，该模块用于支持用户态驱动，如果不使能加速器仅支持内核态驱动

* 虚拟化热迁移依赖的内核config选项有：

  * CONFIG_VFIO=m							vfio.ko编译配置选项
  	 CONFIG_VFIO_PCI=m						vfio_pci.ko编译配置选项

  此外加速器内核态算法的实现依赖于内核crypto子系统。

#### 模块依赖及参数配置指导

* 当用户态驱动依赖的config项CONFIG_UACCE未配置时，ZIP、HPRE、SEC驱动支持内核态。hisi_qm模块为zip、sec、hpre三个加速器设备驱动提供内核态中间层接口，故这几个ko的依赖关系为：
  * hisi_zip.ko《== hisi_qm.ko
  * hisi_sec2.ko《== hisi_qm.ko 
  * hisi_hpre.ko《== hisi_qm.ko
  * hisi-trng-v2.ko《== rng-core.ko
* 加速器模块(SEC、ZIP、HPRE)参数如下表所示，模块参数的支持范围和配置策略会在配置说明中列出。加载驱动时模块参数没有先后顺序。加载驱动后，可以通过cat /sys/bus/pci/drivers/<driver>/module/parameters/ 来查询模块参数。模块参数在加载驱动后不支持更新。如zip驱动的加载（示例为不支持用户态的情况下）：
  * insmod hisi_zip.ko pf_q_num =16 vfs_num=1 sgl_sge_nr=16
* 模块参数说明

| 参数       | 参数范围                                | 加载配置示例                                                 | 配置说明                 |
| ---------- | :-------------------------------------- | ------------------------------------------------------------ | ------------------------ |
| pf_q_num   | 2~1024 SEC 默认为256 HPRE/ZIP 默认为64 | insmod hisi_zip.ko pf_q_num=16<br />insmod hisi_hpre.ko pf_q_num=16<br />insmod hisi_sec2.ko pf_q_num=16 | PF可用队列数             |
| vfs_num    | 0~63 默认值为0                          | insmod hisi_zip.ko vfs_num=1<br />insmod hisi_hpre.ko vfs_num=1<br />insmod hisi_sec2.ko vfs_num=1 | 使能VF数                 |
| sgl_sge_nr | 1~225 默认值为10                        | insmod hisi_zip.ko sgl_sge_nr=16                             | 单个sgl的sge数量 （仅ZIP模块参数） |
| ctx_q_num  | 2~32，仅偶数默认值为2                   | insmod hisi_sec2.ko ctx_q_num=16                             | crypto tfm下的硬件队列数 |
| uacce_mode | 0、1 默认值为0                          | insmod hisi_zip.ko uacce_mode=1<br />insmod hisi_hpre.ko uacce_mode=1<br />insmod hisi_sec.ko <u>uacce_mode=1</u> | 用户态驱动开关           |

### 二、ZIP SEC HPRE内核态特性

#### 1、加速器支持SRIOV

* 特性介绍

  加速器设备支持通过PCI驱动框架注册设备驱动，加速器设备支持PCI function的SRIOV功能实现设备虚拟化操作，并且支持通过驱动模块参数直接配置虚拟化设备VF的个数。

  用户通过vfs_num模块参数，可设置初始的vf数目，加载驱动模块后，每个加速器PF将产生对应数量的vf（不支持模块加载后再对该模块参数进行更新，且通过模块参数只能查询到初始配置的值）。同时，支持pci sysfs方式更新vf的个数，可用于更新对应PCI物理设备的vf数量，配置功能仅在PF，同时pf支持的算法业务、FLR、限流vf同样支持。

* 涉及代码与使能

  | commitid     | subject                                                      | openeuler enabled（Y/N） |
  | ------------ | ------------------------------------------------------------ | ------------------------ |
  | 5ec302a364bf | crypto: hisilicon - add SRIOV support for HPRE               | Y                        |
  | 35ee280fb1fb | crypto: hisilicon - add vfs_num module parameter for hpre/sec | Y                        |
  | 73bcb049a77  | crypto: hisilicon - add SRIOV for HiSilicon SEC              | Y                        |
  | 79e09f30eeba | crypto: hisilicon - add SRIOV support for ZIP                | Y                        |
  | 39977f4b51c  | crypto: hisilicon - add vfs_num module param for zip         | Y                        |

  

* 示例

  用户也可以在加载驱动时直接通过模块参数vfs_num配置vf：

  ~~~
  modprobe hisi_zip vfs_num=1
  ~~~

  模块加载之后支持pci sysfs方式跟新vf个数：

  ~~~
  echo 2 > /sys/devices/<pci_bus>/<pci_device>/<device>/sriov_numvfs
  echo 2  > /sys/devices/pci0000:30/0000:30:00.0/0000:31:00.0/sriov_numvfs
  ~~~

#### 2、加速器支持限流功能

* 特性介绍

  HAC(HPRE/ZIP/SEC)加速器设备在支持虚拟化时，不同VF设备分配给不同的虚拟机VM，即不同的用户使用，考虑到加速器作为附加的硬件计算资源，需要以‘算力’为单位，每个VF的算力应该定量分配出去，让每个用户能够获得足够的计算资源，同时，让同一个加速器物理设备上的VF的‘算力’相互不受干扰，用户购买了多少算力，比如AES-CBC 20Gbps，那么就给可以用户配20Gbps算力。仅支持root权限用户可以配置算力，可对各个function(包括PF) 进行‘算力’大小查询与配置，VF用户只能查询，不能配置。

  用户可以在设备debugfs下面看到alg_qos文件节点，算力配置命令为echo bdf qos > alg_qos,以Hip09为例，用户知道aes-cbc满配是100Gbps，用户想要设置0000:39:00.1配置20Gbps，那么实际上输入为20%，则echo 0000:39:00.1 200 > alg_qos，如果想要配置0000:39:00.2，则echo 0000:76:00.2 200 > alg_qos ，zip支持压缩、解压缩，hpre支持ECC、RSA。sec支持加解密算法配置、qos表示流量比例。从1~1000, 表示0.1%~100.0%。

* 涉及代码与使能

  | commitid     | subject                                                      | openeuler enabled（Y/N） |
  | ------------ | ------------------------------------------------------------ | ------------------------ |
  | 72b010dc33b9 | crypto: hisilicon/qm - supports writing QoS int the host     | Y                        |
  | cc0c40c613d2 | crypto: hisilicon/qm - add the "alg_qos" file node           | Y                        |
  | 362c50bad3a7 | crypto: hisilicon/qm - merges the work initialization process into a single function | Y                        |
  | 2966d9d3078c | crypto: hisilicon/qm - add pf ping single vf function        | Y                        |
  | 3bbf0783636b | crypto: hisilicon/qm - supports to inquiry each function's QoS | Y                        |
  | 3d2a429271bb | crypto: hisilicon/sec - adds the max shaper type rate        | Y                        |

  

* 示例

  sec vf流量配置：

  ~~~
  modprobe hisi_sec2 vfs_num=2
  echo 0000:39:00.1 200 > /sys/kernel/debug/hisi_sec2/0000:39:00.1/alg_qos
  cat /sys/kernel/debug/hisi_sec2/0000:39:00.1/alg_qos
  ~~~

#### 3、加速器提供配置信息查询功能

* 特性介绍

  支持通过DebugFS文件系统，查询当前设备的基本配置信息，错误信息和业务运行时的状态信息。

  DebugFS能够查询到的信息：

  * 首先是基本的状态查询，包含软件和硬件：QM状态查询功能，SQC/CQC配置信息，配置寄存器等信息。
  * 其次，正常和异常下收发包数量统计。
  * 另外包括临终遗言功能，复位前将异常变化的寄存器dump，这些寄存器复位后会被还原。
  * 还包括提供寄存器一键比对check功能，快速发现前后变化的寄存器状态。

* 涉及代码与使能

  | commitid     | subject                                                    | openeuler enabled（Y/N） |
  | ------------ | ---------------------------------------------------------- | ------------------------ |
  | 72c7a68d2ea3 | crypto: hisilicon - add debugfs for ZIP and QM             | Y                        |
  | 8502652542c6 | crypto: hisilicon/qm - add debugfs for QM                  | Y                        |
  | 0a3a3960210b | crypto: hisilicon/qm - add debugfs to the QM state machine | Y                        |
  | c31dc9fe165d | crypto: hisilicon/qm - add DebugFS for xQC and xQE dump    | Y                        |
  | 848974151618 | crypto: hisilicon - Add debugfs for HPRE                   | Y                        |
  | 1e9bc276f    | crypto: hisilicon - add DebugFS for HiSilicon SEC          | Y                        |

  

* 示例

  查看zip设备qm dfx：

  ~~~
  cat /sys/kernel/debug/hisi_zip/0000:31:00.0/qm/regs 查看qm相关寄存器
  cat /sys/kernel/debug/hisi_zip/0000:31:00.0/comp_core0/regs 查看算法核相关寄存器
  ~~~

  更多更详细DFX功能可查看加速器DFX手册。

#### 4、加速器内核态基本算法加速功能

* 特性介绍

  linux内核加/解密和压缩/算法有内核crypto子系统承载，crypto子系统支持内核态算法扩展及替换。加速器内核态驱动在模块初始化阶段将加速器硬算算法接口注册到crypto子系统中，通过crypto相关算法的接口可以调用加速器设备。 

  查看加速器驱动支持的算法

  ~~~
  cat /proc/crypto | grep -B 4 -A 8 hisi_sec2		//查看sec设备支持的算法
  cat /proc/crypto | grep -B 4 -A 8 hisi_hpre		//查看hpre设备支持的算法
  cat /proc/crypto | grep -B 4 -A 8 hisi_zip		//查看zip设备支持的算法
  ~~~

* 涉及代码与使能

  | commitid     | subject                                                      | openeuler enabled（Y/N） |
  | ------------ | ------------------------------------------------------------ | ------------------------ |
  | 62c455ca853e | crypto: hisilicon - add HiSilicon ZIP accelerator support    | Y                        |
  | 7b44c0eecd6a | crypto: hisilicon/sec - add new skcipher mode for SEC        | Y                        |
  | 2f072d75d1ab | crypto: hisilicon - Add aead support on SEC2                 | Y                        |
  | c16a70c1f253 | crypto: hisilicon/sec - add new algorithm mode for AEAD      | Y                        |
  | 6c46a3297bea | crypto: hisilicon/sec - add fallback tfm supporting for aeads | Y                        |
  | c8b4b477079d | crypto: hisilicon - add HiSilicon HPRE accelerator           | Y                        |
  | fbc75d03fda0 | crypto: hisilicon/hpre - enable Elliptic curve cryptography  | Y                        |
  | 05e7b906aa7c | crypto: hisilicon/hpre - add 'ECDH' algorithm                | Y                        |
  | b981f7990e1a | crypto: hisilicon/hpre - register ecdh NIST P384             | Y                        |
  | 90274769cf79 | crypto: hisilicon/hpre - add 'CURVE25519' algorithm          | Y                        |
  | 3e90efd12959 | hwrng: hisi - add HiSilicon TRNG driver support              | Y                        |
  | 6e57871c3b75 | crypto: hisilicon/trng - add version to adapt new algorithm  | Y                        |

* zlib、gzip算法调用

  加载hisi_zip.ko后，通过crypto子系统acomp接口实现压缩解压缩功能。

  接口路径：<linux/include/crypto/acompress.h>

* DH算法调用

  加载hisi_hpre.ko后，通过crypto子系统KPP接口实现密钥协商功能。

  接口路径：<linux/include/crypto/kpp.h>

  规格限制：HPRE硬件设备支持DH算法规格如下表。 

  ​							DH group1/2/5/14/15/16模幂长度

  | group   | 模值(bits) |
  | ------- | ---------- |
  | group1  | 768        |
  | group2  | 1024       |
  | group5  | 1536       |
  | group14 | 2048       |
  | group15 | 3072       |
  | group16 | 4096       |

* RSA算法调用

  加载hisi_hpre.ko后，通过crypto子系统AKCIPHER接口实现加解密与签名功能。

  接口路径：<linux/include/crypto/akcipher.h>

  规格限制：HPRE硬件设备支持RSA算法规格如下表。 

  ​									RSA密钥位宽

  | 模式 | 密钥位宽(bits)         |
  | ---- | ---------------------- |
  | 标准 | 1024、2048、3072、4096 |
  | CRT  | 1024、2048、3072、4096 |

* ECDH算法调用

  加载hisi_hpre.ko后，通过crypto子系统KPP和ECDH接口实现基于椭圆曲线的密钥协商功能。

  接口路径：<linux/include/crypto/kpp.h>

  ​		 < linux/include/crypto/ecdh.h>

  规格限制：HPRE硬件设备支持ECDH算法的椭圆曲线规格如下表所示。

  ​									ECDH椭圆曲线规格

  | 曲线名称            | 密钥位宽(bits) |
  | ------------------- | -------------- |
  | ECC_CURVE_NIST_P192 | 192            |
  | ECC_CURVE_NIST_P224 | 224            |
  | ECC_CURVE_NIST_P256 | 256            |
  | ECC_CURVE_NIST_P384 | 384            |
  | ECC_CURVE_NIST_P521 | 521            |

* X25519算法调用

  加载hisi_hpre.ko后，通过crypto子系统CURVE25519接口实现基于椭圆曲线curve25519的密钥协商功能。

  接口路径：<linux/include/crypto/curve25519.h>

  规格限制：HPRE硬件设备支持X25519算法的椭圆曲线规格如下表所示。

  ​									X25519椭圆曲线规格

  | 曲线名称   | 密钥位宽(bits) |
  | ---------- | -------------- |
  | Curve25519 | 256            |

* SKCIPHER算法调用

  加载hisi_sec2.ko后，通过crypto子系统SKCIPHER接口实现加解密功能。

  接口路径：<linux/include/crypto/skcipher.h>

  规格限制：SEC硬件设备支持SKCIPHER算法规格如下表。

  ​									SKCIPHER算法规格

  | SKCIPHER算法名称 | 设置key长(bits) |
  | ---------------- | --------------- |
  | ecb(aes)         | 128、192、256   |
  | cbc(aes)         | 128、192、256   |
  | xts(aes)         | 256、512        |
  | ecb(des3_ede)    | 192             |
  | cbc(des3_ede)    | 192             |
  | cbc(sm4)         | 128             |
  | xts(sm4)         | 256             |

* AEAD算法调用

  加载hisi_sec2.ko后，通过crypto子系统AEAD接口实现加解密功能。

  接口路径：<linux/include/crypto/aead.h>

  规格限制：SEC硬件设备支持AEAD算法规格如下表。

  ​									AEAD算法规格

  | AEAD算法名称                   | 设置key长(bits) | 设置author key长(bits) | Author长度(bits) |
  | ------------------------------ | --------------- | ---------------------- | ---------------- |
  | authenc(hmac(sha1),cbc(aes))   | 128、192、256   | 0~512                  | 1~160            |
  | authenc(hmac(sha256),cbc(aes)) | 128、192、256   | 0~512                  | 1~256            |
  | authenc(hmac(sha512),cbc(aes)) | 128、192、256   | 0~1024                 | 1~512            |

* drbg算法调用

  加载hisi-trng-v2.ko后，通过crypto子系统drbg接口实现获取随机数功能。

  接口路径：<linux/include/crypto/drbg.h>

  规格限制：只支持seed大小为384bits，最大获取随机数长度为4095*128bits。

* 真随机数获取

  加载hisi-trng-v2.ko后，支持通过/dev/hwrng获取TRNG设备产生的真随机数。用户从/dev/random中获取的随机数有部分是TRNG设备产生的真随机数 

#### 5、加速器支持热迁移功能

* 特性介绍

  加速器热迁移主要配合KVM-QEMU完成直通加速器VF设备的VM和VM之间的动态热迁移，在热迁移的过程中保证业务的不中断和用户不感知。因为openeuler是5.10的内核，没有合入linux主线最新的热迁移框架，当前openeuler使用的是V1版的热迁移（祥见下面patch）适配qemu 6.2.0到8.0之间的版本，qemu 8.x的版本已经不在支持V1版的热迁移。

* 涉及代码与使能

  | commitid     | subject                                                     | openeuler enabled（Y/N） |
  | ------------ | ----------------------------------------------------------- | ------------------------ |
  | a0464f0b70f9 | vfio/hisilicon: add acc live migration driver               | Y                        |
  | e073afaff8c1 | crypto: hisilicon/qm - support the userspace task resetting | Y                        |

* 示例

  zip热迁移：

  ~~~
  modprobe hisi_zip
  创建vf：
  echo 0 > /sys/devices/pci0000:30/0000:30:00.0/0000:31:00.0/sriov_numvfs
  echo 2 > /sys/devices/pci0000:30/0000:30:00.0/0000:31:00.0/sriov_numvfs
  
  绑定vfio:
  echo 0000:31:00.1 > /sys/bus/pci/drivers/hisi_zip/unbind
  echo vfio-pci > /sys/devices/pci0000:30/0000:30:00.0/0000:31:00.1/driver_override
  echo 0000:31:00.1 > /sys/bus/pci/drivers_probe
  
  echo 0000:31:00.2 > /sys/bus/pci/drivers/hisi_zip/unbind
  echo vfio-pci > /sys/devices/pci0000:30/0000:30:00.0/0000:31:00.2/driver_override
  echo 0000:31:00.2 > /sys/bus/pci/drivers_probe
  
  启动迁出端：
  qemu-system-aarch64 -machine virt,gic-version=3 -enable-kvm -cpu host -m 2G -smp 1 \
  -kernel /home/Image -initrd /home/shan/minifs.cpio.gz -nographic \
  -append "rdinit=init console=ttyAMA0 earlycon=pl011,0x9000000 kpti=off cma=2G" \
  -device vfio-pci,host=31:00.1,x-enable-migration=true,x-pre-copy-dirty-page-tracking=off \
  -monitor stdio -serial telnet:0.0.0.0:1111,server,nowait -net none
  
  启动迁入端：
  qemu-system-aarch64 -machine virt,gic-version=3 -enable-kvm -cpu host -m 2G -smp 1 \
  -kernel /home/Image -initrd /home/shan/minifs.cpio.gz -nographic \
  -append "rdinit=init console=ttyAMA0 earlycon=pl011,0x9000000 kpti=off cma=2G" \
  -device vfio-pci,host=31:00.2,x-enable-migration=true,x-pre-copy-dirty-page-tracking=off \
  -monitor stdio -serial telnet:0.0.0.0:2222,server,nowait -net none -incoming tcp:0:6666
  
  迁出端vm加载加速器驱动：
  insmod hisi_zip.ko （可跑加速器业务，目前openEuler VM仅支持内核态业务）
  
  host迁出端控制台启动迁移:
  migrate tcp:0.0.0.0:6666
  ~~~

#### 6、加速器支持RAS复位特性

* 特性介绍

  Reliability—可靠性 、Availability—可用性、Serviceability—可服务性简称RAS，RAS是用于保障HAC设备的可靠性，即当HAC出现硬件错误时，尽最大可能不影响业务连续性与正确性。加速器RAS特性主要包括设备停流、缓存现场、复位、恢复业务。当加速器设备发生硬件错误时会触发RAS中断，在保证用户（UADK接口或者内核Crypto接口的调用者）业务连续性和正确性的前提下，中断处理程序完成对硬件错误的纠正。

* 使涉及代码与能

  | commitid     | subject                                                      | openeuler enabled（Y/N） |
  | ------------ | ------------------------------------------------------------ | ------------------------ |
  | b67202e8ed30 | crypto: hisilicon/qm - add state machine for QM              | Y                        |
  | dbdc1ec31fc0 | crypto: hisilicon - add device error report through abnormal irq | Y                        |
  | 6c6dd5802c2d | crypto: hisilicon/qm - add controller reset interface        | Y                        |
  | 1f5c9f34f0cc | crypto: hisilicon/hpre - add controller reset support for HPRE | Y                        |
  | c4aab24448a3 | crypto: hisilicon - enable new error types for QM            | Y                        |
  | ed278023708b | crypto: hisilicon/hpre - add two RAS correctable errors processing | Y                        |
  | 141876c252a4 | crypto: hisilicon/sec2 - add controller reset support for SEC2 | Y                        |
  | 1db0016e0d22 | crypto: hisilicon/qm - do not reset hardware when CE happens | Y                        |


#### 7、加速器支持FLR复位功能

* 特性介绍

  加速器设备支持通过PCI驱动框架注册设备驱动，提供基于PCI设备的FLR复位功能，完成Function级别的复位，同时实现软件和硬件恢复到初始化的状态，在复位场景下，保证业务流合理停止处理（合理停止：停流时用户会被通知，用户对未完成的任务进行相应处理，用户可以选择，在复位完成后，基于初始状态的function设备继续进行业务，或也可以直接退出业务处理，其中用户是指UADK接口或者内核Crypto接口的调用者） 

* 涉及代码与使能

  | commitid     | subject                                                      | openeuler enabled（Y/N） |
  | ------------ | ------------------------------------------------------------ | ------------------------ |
  | 7ce396fa12a9 | crypto: hisilicon - add FLR support                          | Y                        |
  | 38cd3968bf28 | crypto: hisilicon/qm - adjust reset interface                | Y                        |
  | 7ed83901326f | crypto: hisilicon/qm - add stop queue by hardware            | Y                        |
  | e3ac4d20e936 | crypto: hisilicon/qm - enable PF and VFs communication       | Y                        |
  | 3cd53a27c2fc | crypto: hisilicon/qm - add callback to support communication | Y                        |
  | 760fe22cf5e9 | crypto: hisilicon/qm - update reset flow                     | Y                        |

* 示例

  ~~~
  zip pf reset:
  echo 1 > /sys/bus/pci/devices/0000:31:00.0/reset
  
  zip vf reset:
  echo 1 > /sys/bus/pci/devices/0000:31:00.1/reset
  ~~~

#### 8、加速器设备支持低功耗特性

* 对于加速器设备，硬件上支持对单个PF设备进行电源的打开与关闭。用户可以通过Device SysFS(/sys/devices/.../power/control)对单个PF设备进行低功耗控制配置，如果使能‘auto’策略，设备将在被使用时（内核与用户态应用在调用Crypto或Warpdrive接口获取硬件资源）自动进入运行状态，如果使能了‘on’策略，那么该设备就一直处于运作状态，不进行任何功耗控制。由于PCIE协议要求在D3cold电源状态下VF不应该使能起来，因此在使能VF时不进入下电状态，此时依赖芯片本身支持的clock-gating来降低功耗。

* 涉及代码与使能

  | commitid     | subject                                                      | openeuler enabled（Y/N） |
  | ------------ | ------------------------------------------------------------ | ------------------------ |
  | d7ea53395b72 | crypto: hisilicon - add runtime PM ops                       | Y                        |
  | 607c191b371d | crypto: hisilicon - support runtime PM for accelerator device | Y                        |
  | ed5fa39fa8a6 | crypto: hisilicon - enable zip device clock gating           | Y                        |
  | 3d845d497b23 | crypto: hisilicon - enable sec device clock gating           | Y                        |
  | ea5202dff79c | crypto: hisilicon - enable hpre device clock gating          | Y                        |

* 示例

  zip设备配置低功耗策略

  ~~~
  cat /sys/bus/pci/devices/0000:31:00.1/power/runtime_status
  echo auto > /sys/bus/pci/devices/0000:31:00.1/power/control
  cat /sys/bus/pci/devices/0000:31:00.1/power/runtime_status
  ~~~

#### 9、支持获取硬件随机数特性 

* 特性介绍

  芯片TRNG模块提供硬件随机数算法，内核TRNG驱动（drivers/crypto/hisilicon/trng）提供Crypto RNG算法，往Kernel Crypto子系统进行注册，同时注册TRNG真随机数设备驱动到HWRANDOM子系统。这样，通过内核CRYPTO RNG接口与/dev/random接口可分别使用芯片TRNG所提供的DRBG与TRNG真随机数。

* 涉及代码与使能

  | commitid     | subject                                                     | openeuler enabled（Y/N） |
  | ------------ | ----------------------------------------------------------- | ------------------------ |
  | 3e90efd12959 | hwrng: hisi - add HiSilicon TRNG driver support             | Y                        |
  | 6e57871c3b75 | crypto: hisilicon/trng - add version to adapt new algorithm | Y                        |

  

* 示例

  ~~~
  cat /dev/hwrng
  cat /dev/random
  ~~~

  

## UADK 模块介绍
* 详情可以参考UADK用户手册 https://gitee.com/openeuler/uadk/wikis/%E4%BD%BF%E7%94%A8%E6%96%87%E6%A1%A3/UADK%20quick%20start
