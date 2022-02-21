# 2022 SP Promotion

时间2月中旬-6月中旬，学习scope需要包含以下内容：

* ARM64
* Linux kernel
* Compiler linker

设备以树莓派4b设备为主（[Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/))，穿插学习法，ARM64和Linux内容有很大的相关性.

### ARM64

总共40.5h，计划每天2h的视频理论学习时间50h（包含笔记时间），需要25天时间

* [x] 课程介绍
* [x] ARM处理器介绍和ARMv8概述
* [x] 使用QEMU+树莓派4+JLINK搭建调试环境
* [x] ARM64汇编：存储指令
* ARM64汇编：算数和移位
* ARM64汇编：比较和跳转
* ARM64汇编：其他重要指令
* ARM64汇编：汇编一些易错点
* GUN AS汇编器介绍
* LD链接器
* 内嵌汇编
* ARM64异常处理
* ARM64异常处理之终端处理
* GIC中断控制器
* ARMv8内存管理
* MMU实验讲解
* cache和TLB基础知识
* cache一致性-part1
* 解读armv8芯片手册cache
* cache实验讲解
* TLB基础知识
* 内存屏障基础知识
* 解读armv8手册中的内存屏障
* 缓存一致性和内存屏障
* 原子操作
* 浮点指令
* NEO适量优化
* 可扩展矢量计算SVE/SVE2
* GICv3中断控制器和ITS服务
* 深入理解SMMUv3和IOMMU驱动
* SMMU之SVA
* 深入理解AXI5以及AXI5-Lite总线
* 深入理解ACE以及ACE-Lite总线

### Linux Kernel

* 奔跑吧Linux内核 
* Linux内核视频

### Compiler Linker

* 静态：编译和链接
* 静态：目标文件里都有什么
* 静态链接
* PE/COFF-windows格式
* 动态：动态装载与进程
* 动态链接
* Linux共享库的组织
* windows的动态链接
* 内存
* 运行库
* 系统调用API
* 运行库的实现

## 计划表单

  

| Time Line    | ARM64                                 | Linux Kernel   | Linker                 | 实验环境 |
| ------------ | ------------------------------------- | -------------- | ---------------------- | -------- |
| 9-Feb        | * 课程介绍                            | 内核简介       |                        |          |
| 10-Feb       | * ARM处理器介绍和ARMv8概述            | 内核出发       |                        |          |
| 11/12/13-Feb | *  使用QEMU+树莓派4+JLINK搭建调试环境 |                |                        |          |
| 2.14-2.16    | * ARM64汇编：存储指令                 |                | 编译与链接             |          |
|              | * ARM64汇编：算数和移位               |                | 目标文件有什么         |          |
|              | * ARM64汇编：比较和跳转               |                | 静态链接               |          |
|              | * ARM64汇编：其他重要指令             |                | 可执行文件的装载和进程 |          |
|              | * ARM64汇编：汇编一些易错点           |                | Linux共性库的组织      |          |
|              | * GUN AS汇编器介绍                    | 中断和异常     |                        |          |
|              | * LD链接器                            | 内存寻址       | 内存                   |          |
|              | * 内嵌汇编                            | 内存管理       |                        |          |
|              | * ARM64异常处理                       |                |                        |          |
|              | * ARM64异常处理之终端处理             |                |                        |          |
|              | * GIC中断控制器                       |                |                        |          |
|              | * ARMv8内存管理                       |                |                        |          |
|              | * MMU实验讲解                         |                |                        |          |
|              | * cache和TLB基础知识                  | 内核同步       |                        |          |
|              | * cache一致性-part1                   | 进程           |                        |          |
|              | * 解读armv8芯片手册cache              | 进程调度       |                        |          |
|              | * cache实验讲解                       | 进程通信       |                        |          |
|              | * TLB基础知识                         | 进程地址空间   |                        |          |
|              | * 内存屏障基础知识                    |                |                        |          |
|              | * 解读armv8手册中的内存屏障           |                |                        |          |
|              | * 缓存一致性和内存屏障                |                |                        |          |
|              | * 原子操作                            | 定时测量       |                        |          |
|              | * 浮点指令                            | 系统调用       | 动态链接               |          |
|              | * NEO适量优化                         | 信号           | 运行库                 |          |
|              | * 可扩展矢量计算SVE/SVE2              | 虚拟文件系统   | 系统调用和API          |          |
|              | * GICv3中断控制器和ITS服务            | IO体系设备驱动 | 运行库的实现           |          |
|              | * 深入理解SMMUv3和IOMMU驱动           | 块设备驱动     |                        |          |
|              | * SMMU之SVA                           | 页高速缓存     |                        |          |
|              | *  深入理解AXI5以及AXI5-Lite总线      | 访问文件       |                        |          |
|              | * 深入理解ACE以及ACE-Lite总线         |                |                        |          |

## 参考资料

1. ARM手册： https://developer.arm.com/documentation/ddi0487/latest

2. 工具： https://github.com/NaoTu/DesktopNaotu/releases/tag/v3.2.3