
# openEuler WayCa 鲲鹏xxx xxx用户使用指南

## 使用场景

- 存储、ceph、待探索 （NA或介绍各场景收益及使用特性）

## 硬件环境
- 鲲鹏920
- 固件依赖需求可按照实际情况补齐 bmc cpld bios 最低版本要求。

## 软件版本

- openEuler 22.03 lts
- openEuler 22.03 lts SP1

## 安装使用

- 1）安装openEuler系统
-- 详见安装指导
- 2）本地ISO 源配置
-- 挂载BMC iso
-- mount xxx
-- 修改 sourcelist
-- rpm update
- 3）安装dpdk软件包
-- rpm install xxx
-- xxx -v
- 4）基本功能测试
-- testpm xxx
-- xxx效果
备注：保证功能可用

## 交流答疑

- https://gitee.com/openeuler/wayca/issue
提单title 标识： 【Way-Kunpeng-高速网络-dpdk】

备注：用于交流基本使用、特性、场景、需求和问题答疑等。