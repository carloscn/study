# ARMv8

## Introduction

* 新一代64位处理
* 保持ARMv7兼容性

## new feature

在programmer guide 2.1里面 引入那些feature：

* Large physical address 

  32位系统的没有enable的话，只支持4G。

* 64bit virtual addressing

  使之虚拟地址空间可以超过4GB

* automatic event sinaling

  支持原子操作的存储和访问的操作

* larger register files

  减少对栈的使用，提高性能。

* ...

* Addtional 16KB 和64KB TLB

* ...

* Load-Acquire, Store Release instructions

* NEON double-precision floating-points

## ARMv8基本概念

手册 RM手册：

* PE： processing element.

RISC架构的特性（RM提供的）：

*  A large uniform register file.
* A load/store architecture, where data-processing operations only operate on register contents, not directly on memory contents.
* Simple address modes, with all the load/store address determined from register contents and instruction fields only. （采用内存映射MMU模式）

### 执行状态Execution state

ARMv8如何向后兼容呢？RM提供 A1.3.

