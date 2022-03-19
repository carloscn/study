# 04_ARMv8指令集-运算指令集

* 加法指令ADD、ADDS、ADCS
* 减法指令SUB、SUBS、SBC，SBCS，CMP
* 位操作AND, ANDS, ORR、EOR、BFI、UBFX、SBFX

## 1. 加法指令

加法指令有ADD、ADDS、ADCS。 ADD一般性加法指令，ADCS带C标志位运算的加法指令，ADDS影响C标志位的加法运算。

### 1.1 ADD

a = a + b, 没有进位标志，也不会利用进位标志

* ADD (extended register) :
  * Define:  `ADD <Xd|SP>, <Xn|SP>, <Wm>, {<extend> {#<amount>}}`
  * Example1: `add x0, x1, x2 ` ( x0 = x1 + x2 )
  * Example2: `add x0, x1, x2, lsl #5`( x0 = x1 + (x2 << 5) )
* ADD (immediate):
  * Define:  `ADD <Xd|SP>, <Xn|SP>, #<imm>{, lsl <#shift>}`, **note shift supports #0 and #12 only**.
  * Example1: `add x1, x2, #8 ` (x1 = x2 + 8)
  * Example2: `add x1, x2, #8, lsl #12` ( x1 = x2 + (8 << 12) )
* ADD (shifted register): 
  * Define: `ADD <Xd>, <Xn>, <Xm>{, <shift> #<amount>}` , note #amount range 0 to 63
  * **Note:  LSL when shift = 0, LSR when shift = 1, ASR when shift = 2**
  * Example1: `add x1, x2, x3, asr #2 `

### 1.2 ADDS

(a,C) = a + b, 带进位标志的加法，用法和ADD一样

### 1.3 ADCS

(a,C) = a + b + C，带进位标志的加法，且需要加上C标志位，用法和ADD一样。 注意，如果加法溢出的时候C标志位会置位为1，比如，a = 0xFFFFFFFFFFFFFFFF, b = 1，此时，加法溢出，C置位1。

### 1.4 ADR

a = b + PC, 当前程序的PC值加上给定的地址偏移

* ADR
  * Define: `ADR <Xd>, <label>`
  * Note, no 32-bit
  * Note,  <label>  range ±1MB, offset from the address of this instruction.
  * Example01: `adr x1, #25`

### 1.5 关于查看C flag的方法

#### 方法1：使用MSR/MRS指令

```assembly
	msr NZCV, xzr	   // clear the NZCV
	mrs x0, NZCV     // 查看NZCV寄存器，NZCV在高位28 - 32 bits
```

#### 方法2：使用ADCS的+C特性

`adcs x0, zxr, xzr`  让两个0寄存器相加 0+0+c就可以得到C标志位的值

## 2. 减法指令

减法指令包含SBC，SBCS。请参考ARMv8手册，C6.2.231   C6-1299

### 2.1 SUB

a = a - b, 没有进位标志，也不会利用进位标志。使用方法和ADD一致。

### 2.2 SUBS

(a,N) = a - b, 会置标志位N。使用方法和SUBS一致，减成负数的时候，其余位置补1。

### 2.3 SBC

a = a - b - 1 + C

* SBC (Subtract with Carry):
  * Define: `SBC <Xd>, <Xn>, <Xm>`
  * Example: `sbc x0, xzr, xzr` 

### 2.4  SBCS

(a, N) = a - b - 1 + C， 如果减出负数的话，N会被置位

* SBC (Subtract with Carry, setting N flag):
  * Define: `SBCS <Xd>, <Xn>, <Xm>`
  * Example: `sbcs x0, xzr, x1` 

### 2.5 [CMP](https://developer.arm.com/documentation/ddi0597/2021-12/Base-Instructions/CMP--immediate---Compare--immediate--?lang=en)

比较指令，实际上也使用SBC实现的, cmp x1, x2 

* 若x1 > x2, NCZV = 0100
* 若x1 = x2, NCZV = 0110
* 若x1 < x2, NCZV = 1000

Define 1: `CMP <Xn|SP>, <R><m>{, <extend> {#<amount>}}`

Define 2: `CMP <Xn|SP>, #<imm>{, <shift>}`

Define 3: `CMP <Xn|SP>, <Xm>{, <shift> #<amount>}`

Example:

 ** The function cmp_and_return_test:*

 ** if a >= b return 1*

 ** if a < b return 0*

```assembly
test_cmp:
	cmp x0, x1   			// if x0 >= x1,  C is 1; if x0 < x1 C is 0
  adcs x0, xzr, xzr // 0 + 0 + C
```

## 3. 位操作

位操作包含AND, ANDS, ORR、EOR、BFI、UBFX、SBFX， 分别是与、与置位标志位、或、异或、插入、无符号提取、有符号提取。

### 3.1 ORR

a = a | b;

Define 1: `ORR <Xd|SP>, <Xn>, #<imm>`

Define 2: `ORR <Xd|SP>, <Xn>, <Xm>{, <shift> #<amount>}`

```assembly
test_orr:
	// ORR test  0xAA oor 0x55 = 0xFF
	//           0xFF oor 0x00 = 0xFF
	//           0xFF oor 0xFF = 0xFF
	//           0x00 oor 0x00 = 0x00
	mov x0, xzr
	mov x1, #0xAA
	mov x2, #0x55
	orr x1, x1, x2

	mov x1, #0xFF
	orr x1, x1, xzr

	mov x1, #0xFF
	orr x1, x1, x1

	orr x1, xzr, xzr

	ret



test_ubfx:
	// x1: 0000 0000 0000 0000  ->  0000 0000 0000 1111
	//                     ^
	//                     |
	// x2:      0000 0000 1111 0000
	mov x1, xzr
	mov x2, #0x00F0
	ubfx x1, x2, #0x4, #0x4

	// x1: 0000 0000 0000 0000  ->  1111 1111 1111 1111
	//                     ^
	//                     |
	//          1000 0000 1111 0000
	mov x1, xzr
	mov x2, #0x80F0
```

### 3.2 EOR

a = a ^ b;

Define 1: `EOR <Xd|SP>, <Xn>, #<imm>`

Define 2: `EOR <Xd|SP>, <Xn>, <Xm>{, <shift> #<amount>}`

```assembly
test_eor:
	// test 2 exchange the value x1 = 0x07, x2 = 0xAA
    // using the orr, just use two register.
	// x1 = x1^x2
	// x2 = x2^x1
	// x1 = x1^x2
	ldr x1, =0x07
	ldr x2, =0xAA
	eor x1, x1, x2
	eor x2, x2, x1
	eor x1, x1, x2
	ret
```

几个EOR的小技巧：

* 翻转某些位: 比如把右数第0位到第3位翻转：  1010 1001 ^ 0000 1111 = 1010 0110 
* 交换数值：  a=a^b;  b=b^a; a=a^b，不借助第三个变量
* 置0： a^a
* 判断相等 a^b == 0

### 3.3 AND

#### 3.3.1 AND

a = a & b;

Define 1: `AND <Xd|SP>, <Xn>, #<imm>`

Define 2: `AND <Xd|SP>, <Xn>, <Xm>{, <shift> #<amount>}`

```assembly
	msr NZCV, xzr	   // clear the NZCV
	ldr x1, =0xAA
	ldr x2, =0x0
	// test AND, no Z flag. x1 = x1&x2
	and x1, x1, x2
	mrs x0, NZCV
```

#### 3.3.2 ANDS 

(a, z) = a & b. 如果a和b与的结果为0，z flag置位

```assembly
	// test ANDS, z flag, if the result is 0, Z is 1
	msr NZCV, xzr	   // clear the NZCV
	mov x0, xzr
	ldr x1, =0xAA
	ands x1, x1, x2
	mrs x0, NZCV
```

### 3.4 BFI

Define 1: `BFI <Xd>, <Xn>, #<lsb>, #<width>`

从Xn寄存器里面从低位开始，插入到Xd寄存器从#<lsb>开始，#<width>长度。

读取Xn寄存器的低位开始计算，插入到Xd寄存器从#<lsb>开始，#<width>长度。这个没有办法控制Xd的位置，只能从Xd的最低位开始。

```assembly
test_bfi:
	// 0000 0000 0000 1010
	//                 |
	//                 V
	//           0000 0000 0000 0000  ->  0000 1010 0000 0000
	ldr x1, =0x000A
	mov x2, xzr
	bfi x2, x1, #0x8, #0x4

	// 0000 0000 0000 1010
	//                 |
	//                 V
	//           0000 0101 0000 0000  ->  0000 1010 0000 0000
	ldr x1, =0x000A
	mov x2, #0x0500
	bfi x2, x1, #0x8, #0x4

	ret
```

### 3.5 UBFX/SBFX

Define: `UBFX <Xd>, <Xn>, #<lsb>, #<width>`

Define: `SBFX <Xd>, <Xn>, #<lsb>, #<width>`

读取Xn寄存器的#<lsb>开始，#<width>长度开始计算，替换到Xd寄存器低位的位置#<width>长度。这个没有办法控制Xd的位置，只能从Xd的最低位开始。 SBFX是有符号的，替换之后Xd 其他位0变为1。

```assembly
	// x1:      0000 0000 1111 0000
	//                     |
	//                     V
	// x2: 0000 0000 0000 0000  ->  0000 0000 0000 1111

	mov x1, #0x00F0
	mov x2, xzr
	ubfx x2, x1, #0x4, #0x4

	//          1000 0000 1111 0000
	//                     |
	//                     V
	// x1: 0000 0000 0000 0000  ->  1111 1111 1111 1111
	mov x1, #0x80F0
	mov x2, xzr
	sbfx x2, x1, #0x4, #0x4
```

![image-20220319154625842](/Users/carlos/workspace/work/study-2022/ARM/_media/image-20220319154625842.png)

# Ref 

[1] [Arm Armv8-A Architecture Registers-NZCV, Condition Flags](https://developer.arm.com/documentation/ddi0595/2021-06/AArch64-Registers/NZCV--Condition-Flags)

[2] [ARM Cortex-A Series Programmer's Guide for ARMv8-A - Arithmetic and logical operations](https://developer.arm.com/documentation/den0024/a/The-A64-instruction-set/Data-processing-instructions/Arithmetic-and-logical-operations?lang=en)

[3] [ARM架构（三）ARMv8 Programm Model Overview](https://willendless.github.io/%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/2021/03/11/ARM%E6%9E%B6%E6%9E%843/)

[4] [ARMv8官方手册学习笔记(三)：寄存器](https://zhuanlan.zhihu.com/p/111790975)