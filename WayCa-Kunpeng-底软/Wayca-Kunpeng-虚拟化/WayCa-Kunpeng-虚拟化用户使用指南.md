# openEuler WayCa 鲲鹏虚拟化用户使用指南
## 软件版本
- openEuler 22.03 lts sp1
- openEuler 22.03 lts sp2
- 用户态软件包：https://gitee.com/src-openeuler/qemu.git

## 工具安装使用
- 安装qemu工具：

yum install qemu-system-aarch64

- qemu使用帮助：

qemu-system-aarch64 -h 可以查看起虚拟机可选的一些参数

## 起虚拟机基本命令

```c
qemu-system-aarch64 -machine virt,gic-version=3 \
-enable-kvm -cpu host -m 4G \
-smp 4 \
-kernel /path/to/Image -initrd /path/to/rootfs -nographic \
-append "rdinit=init console=ttyAMA0 earlycon=pl011,0x90000000" -net none
```
## 虚拟机基本操作

- 退出虚拟机方法

1.按 Ctrl A 松开后迅速按下 C 进入qemu monitor中，输入 q 退出虚拟机；

2.在host(主机)上查询qemu进程：ps -ef | grep qemu-system-aarch64 ，kill 掉对应的进程号虚拟机退出。
- qemu monitor控制台

在qemu monitor控制台中，可以完成很多常规操作，比如设备的增加删除、信息查询、获取虚拟机运行状态等。按 Ctrl A 松开后迅速按下 C 进入qemu monitor中，使用同样的方法退出qemu monitor进入虚拟机。在qemu monitor中输入help可以获取更多指令及其功能信息。

## 其他方法起虚拟机
还可通过libvirt来起虚拟机。
- 工具安装
```c
yum install libvirt
yum install qemu
yum install qemu-img
yum install edk2-aarch64
```
- 启动libvirt服务

systemctl start libvirtd
- 安装虚拟机

1.创建虚拟机img
```c
#硬盘空间大小为5G
qemu-img create -f qcow2 openEuler-image.qcow2 5G
```
2.编写虚拟机安装引导的xml文件

3.将iso拷贝到/var/lib/libvirt/images/目录下

4.定义虚拟机
```c
virsh define openEulerVM.xml
```
5.安装虚拟机
```c
virsh start openEulerVM --console /*openEulerVM为xml中定义的虚拟机名称*/
```
6.其他virsh管理虚拟机命令
```c
virsh list --all /*列出虚拟机列表*/
virsh shutdown openEulerVM /*关闭虚拟机*/
virsh destroy openEulerVM /*立即下电*/
/* 按Ctrl ] 退出虚拟机，虚拟机后台运行 */
virsh console openEulerVM /*接入虚拟机*/
virsh undefine openEulerVM --nvram /*删除虚拟机，使用该命令需确保虚拟机处于关闭状态*/
```