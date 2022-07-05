## Introduction

* 新一代64位处理
* 保持ARMv7兼容性

## New feature

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

## ARMv8 some basic concepts

### RM  datasheet：

* PE： Processing Element（处理机）

#### RISC架构的特性（RM提供的）：

*  A large uniform register file. （ARMv7提供R0-R15，ARMv8提供更丰富的寄存器，比如X0-X30）

* A load/store architecture, where data-processing operations only operate on register contents, not directly on memory contents. 

* Simple address modes, with all the load/store address determined from register contents and instruction fields only. （采用统一的简单的，比如内存映射MMU模式）

### Execution States

The downward compatibility of ARMv7 shall be considered when the ARM designed the ARMv8 instruction set architecture，so the ARM designs the `Execution State` to  be compatible with the ARMv7. There are the two types of `Execution State designed:

* AArch64
* AArch32

The two kinds of `Execution State` are like the different containers for different execution envs. In the lesson, the AArch64 execution state should be focused only. The difference between the AArch64 and the AArch32 exection states are showed as following table. 

| Features                             | AArch64                                                      | AArch32                                                      |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| General-purpose registers            | 31-64bits (X30 is used as the procedure link register)       | 13-32bits registers                                          |
| Program Counter (PC)                 | one 64-bits registers                                        | one 32-bits register                                         |
| Stack Pointers (SP)                  | mulitple 64-bits registers                                   | one 32-bits register                                         |
| Exception Link Register (ELR)        | mulitple 64-bits registers                                   | one 32-bits LR register use as ELR                           |
| Procedure Link Register              | one 64-bits, it is X30 register                              | Sharing the LR with the ELR as PLR.                          |
| Advanced SIMD  vector/floating-point | 32 128-bits registers                                        | 32 64-bits registers                                         |
| Instruction Set                      | A64                                                          | A32 and T32                                                  |
| Excepiton model                      | 4 excepiton levels, EL0-EL3 (ARMv8 Exception Model)          | PE modes and maps this onto the Armv8 Exception model (ARMv7  exception Model) |
| Virtual Addressing                   | 64 bits virual address                                       | 32 bits virtual address                                      |
| Process State (PSTATE)               | The A64 includes instructions that operate diectly on various  PSTATE. | Constract with the AArch64, the A32 and T32 operate them  directly and can also use the APSR, CPSR to access. |

### Instruction Sets

* A64
* A32/T32

#### AArch64 (A64)

Uses 32-bit instruction encodings and fixed-length.

#### AArch32 (A32/T32)

A32 uses 32-bit instruction encodings samely, and fixed-length. However, the T32 is a variable-length instruction set that uses both 16-bit and 32-bit instruction encodings.

Note, in AArch32, the A32/T32 instruction sets were called `ARM and Thumb` instruction sets. The ARMv8 instrcution set extends each of these instruction sets.

### System Registers

* Format
* Types [D13 chapter in RM]

### Supported Data Types

* Byte 8bits
* Halfword 16 bits
* Word 32 bits
* Doubleword 64 bits
* Quadword 128 bits

### Exception Levels (RM-D1.1)

* EL0
* EL1
* EL2
* EL3

<img width="1178" alt="image-20220210211952137" src="https://user-images.githubusercontent.com/16836611/159121502-e03e558f-d920-4b39-832e-e9b36cc0335b.png">

### AArch64's Registers

* R0-R30
* SP
* PC
* V0-V31

![image-20220630171323469](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220630171323469.png)

#### R0-R30

31 general-purpose registers. Each register can be accessed as X0-X30 (all the 64-bit are used) in 64 bit mode, W0-W30 (only low 32-bit are used). The Wx are Xx low 32-bit.

#### SP

64-bit dedicated Stack Pointer register. 

#### PC

A 64-bit Program Counter holding the address of the current instruction.  Software cannot write directly to the PC. It can only be updated on a branch, exception entry or exception return.

### Processor State (RM-D1.7)

#### Condition Flags

* N
* Z
* C
* V

#### Exception Masking bits

* D
* A
* I
* F

#### Exection Status Control bits

* SS
* IL
* nRW
* EL
* SP

