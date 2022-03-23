# 05-ARMv8-指令集-跳转和比较指令

* 零计数指令CLZ
* 比较指令：CMP, CMN
* 跳转指令：B, BR, BL, BLR
* 条件选择指令：CSEL, CSET, CSINC

## 1. CLZ

计算最高为1的比特位前面有多少个0。例如， 0x0800 0000 0000 000F，前面是有4个0的（使用64位的寄存器），如果使用Wn寄存器，按照32位算。

* CLZ
  * Define:  `CLZ <Xd>, <Xn>`
  * Example1: `clz x0, x1 ` [计算x1寄存器内的值最高位1的比特位前面有多少个0，并放入x0]

## 2. 比较指令

比较指令有CMP和CMN，CMP本质为：`CMP x1, x2` -> `x1 = x1 - x2`；CMN为负向比较:`CMN x1, x2`->`x1 = x1 + x2`。

### 2.1 CMP

NZC = a - b，根据不同的比较结果，来确定NZC的标志位，若x1 > x2, NCZV = 0100，若x1 = x2, NCZV = 0110，若x1 < x2, NCZV = 1000。

* CMP (immediate):
  * Define:  `CMP <Xd|SP>, #<imm>{, lsl <#shift>}`, **note shift supports #0 and #12 only**.
  * Example1: `cmp x1, #8 ` (x1 = x2 - 8)
  * Example2: `cmp x1, #8, lsl #12` ( x1 = x2 -  (8 << 12) )
* CMP (shifted register): 
  * Define: `CMP <Xd>, <Xm>{, <shift> #<amount>}` , note #amount range 0 to 63
  * **Note:  LSL when shift = 0, LSR when shift = 1, ASR when shift = 2**
  * Example1: `cmp x1, x2, asr #2 `

### 2.1 CMN

NZC = a + b，a和b如果加和为负数，N置位; a + b的和为0，Z置位；a + b溢出（两个负数相加），C置位。

* CMN (extended register) :
  * Define:  `CMN <Xd|SP>, <R><m>, {<extend> {#<amount>}}`
  * Example1: `cmn x0, x1 ` ( x0 = x0 - x1 )
  * Example2: `cmn x0, x1, lsl #5`( x0 = x0 - (x1 << 5) )
* CMN (immediate):
  * Define:  `CMN <Xd|SP>, #<imm>{, lsl <#shift>}`, **note shift supports #0 and #12 only**.
  * Example1: `cmn x1, #8 ` (x1 = x2 - 8)
  * Example2: `cmn x1, #8, lsl #12` ( x1 = x2 -  (8 << 12) )
* CMN (shifted register): 
  * Define: `CMN <Xd>, <Xm>{, <shift> #<amount>}` , note #amount range 0 to 63
  * **Note:  LSL when shift = 0, LSR when shift = 1, ASR when shift = 2**
  * Example1: `cmn x1, x2, asr #2 `

对于CMN，根据实验有以下几种情况：

| Condition     | Algo                        | N    | Z    | C    |
| ------------- | --------------------------- | ---- | ---- | ---- |
| -0x01 > -0x0E | (-0x01) + (- 0x0E)  = -0x0F | 1    | 0    | 1    |
| 0 = 0         | 0 + 0 = 0                   | 0    | 1    | 0    |
| -0x0F < -0x01 | (-0x0F) + (-0x01) = -0x10   | 1    | 0    | 1    |
| 0x0F > -0x01  | (0x0F) + (-0x01) = 0x0E     | 0    | 0    | 1    |
| -0x0F < 0x01  | (-0x0F) + (0x01) = -0x0E    | 1    | 0    | 0    |
| -0x01 < 0x01  | (-0x01) + 0x01 = 0          | 0    | 1    | 1    |

### 2.3 Condition Codes

(`CMP` and `CMN`) Sets the condition flags to the result of a comparison if the original condition is true. If not true, the conditional flags are set to a specified condition flag state. The conditional compare instruction is very useful for expressing nested or compound comparisons.

Condition codes: [^1]

| Code | Encoding | Meaning (when set by CMP)                            | Meaning (when set by FCMP)                              | Condition flags      |
| ---- | -------- | ---------------------------------------------------- | ------------------------------------------------------- | -------------------- |
| EQ   | `0b0000` | Equal to.                                            | Equal to.                                               | Z =1                 |
| NE   | `0b0001` | Not equal to.                                        | Unordered, or not equal to.                             | Z = 0                |
| CS   | `0b0010` | Carry set (identical to HS).                         | Greater than, equal to, or unordered (identical to HS). | C = 1                |
| HS   | `0b0010` | Greater than, equal to (unsigned) (identical to CS). | Greater than, equal to, or unordered (identical to CS). | C = 1                |
| CC   | `0b0011` | Carry clear (identical to LO).                       | Less than (identical to LO).                            | C = 0                |
| LO   | `0b0011` | Unsigned less than (identical to CC).                | Less than (identical to CC).                            | C = 0                |
| MI   | `0b0100` | Minus, Negative.                                     | Less than.                                              | N = 1                |
| PL   | `0b0101` | Positive or zero.                                    | Greater than, equal to, or unordered.                   | N = 0                |
| VS   | `0b0110` | Signed overflow.                                     | Unordered. (At least one argument was NaN).             | V = 1                |
| VC   | `0b0111` | No signed overflow.                                  | Not unordered. (No argument was NaN).                   | V = 0                |
| HI   | `0b1000` | Greater than (unsigned).                             | Greater than or unordered.                              | (C = 1) && (Z = 0)   |
| LS   | `0b1001` | Less than or equal to (unsigned).                    | Less than or equal to.                                  | (C = 0) \|\| (Z = 1) |
| GE   | `0b1010` | Greater than or equal to (signed).                   | Greater than or equal to.                               | N==V                 |
| LT   | `0b1011` | Less than (signed).                                  | Less than or unordered.                                 | N!=V                 |
| GT   | `0b1100` | Greater than (signed).                               | Greater than.                                           | (Z==0) && (N==V)     |
| LE   | `0b1101` | Less than or equal to (signed).                      | Less than, equal to or unordered.                       | (Z==1) \|\| (N!=V)   |
| AL   | `0b1110` | Always executed.                                     | Default. Always executed.                               | Any                  |
| NV   | `0b1111` | Always executed.                                     | Always executed.                                        | Any                  |

```assembly
// The test is:
// suppose the x1 = 1, x2 = -3
// Use the `cmn` instruction to compare the x1 and x2
// When the result is neg number, make x2 += 1;
// until the result is zero, then return the function.
test_cmn_jump:
	msr NZCV, xzr
	mov x0, xzr
	mov x1, #0x0
loop:
	add x1, x1, #0x1
	mov x2, #-0x3
	cmn x1, x2		// NZCV = 1000
	mrs x0, NZCV
	b.mi loop
	ret
```

## 3. 条件选择指令

条件选择指令包括CSEL\CSET\CSINC三条指令，也是非常重要的指令。

### 3.1 CSEL

CSEL, 条件选择指令，如果cond为真，那么xd就是xn的值，否则就是xm的值。注意，<cond>为上面表格的值。

* Define:  `CSEL <Xd>, <Xn>, <Xm>, <cond> `
* Example1: `csel x1, x1, x2, EQ ` -> 如果Z=1，x1维持原始值，否则x2给x1赋值
* Example2: `csel x1, x1, x2, MI ` -> 如果N=1，x1维持原始值，否则x2给x1赋值

### 3.2 CSET

CSET,  条件置位指令，如果cond为真，那么xd就是1，否则就是0。注意，<cond>为上面表格的值。

* Define:  `CSET <Xd>, <cond> `
* Example1: `cset x1, EQ ` -> 如果Z=1，x1是1， 否则为0
* Example2: `cset x1, MI ` -> 如果N=1，x1是1， 否则为0

### 3.3 CSINC

CSINC，条件选择并增加指令 如果cond为真，那么xd就是xn的值，否则就是xm+1的值。注意，<cond>为上面表格的值。

* Define:   `CSINC <Xd>, <Xn>, <Xm>, <cond> `
* Example1: `csinc x1, x1, x2, EQ ` -> 如果Z=1，x1维持x1， 否则为x2+1
* Example2: `csinc x1, x2, x3, MI ` -> 如果N=1，x1为x2， 否则为x3+1

### 3.4 Example

用汇编实现下面的c代码：

```c
unsigned long cel_test(unsigned long a, unsigned long b)
{
  if (a == 0) {
    return b + 2;
  } else {
    return b - 1;
  }
}
```

分析拆解：

* 比较指令 a与0 的比较，a为0的时候 Z标志位可以使用，因此可以利用Z标识位条件EQ作为该分支的入口。
* b + 2和b-1 分支分别有个运算操作数的动作，确定 +2 还是 -1 可以把 2和 -1 放在一个寄存器里面，由 csel来判断那个值。

```assembly
test_csel:

	mov x2, #0x2
	mov x3, #-0x1
	cmp x0, #0
	csel x4, x2, x3, EQ
	add x0, x1, x4
	ret
```

## 4. 跳转与返回指令

* B： b跳转指令, b  可以跳到PC ±128MB的范围， 不返回
* B. <cond> : 使用跳转指令`b.<cond>`,xx为以上表格的指，不返回
* BR: 跳转到寄存器指定的地址处，不返回
* BL：带返回地址的，PC±128MB， 用于跳转到子函数，返回地址为PC+4，设置到X30寄存器中。
* BLX: 跳转到寄存器指定的地址处，可以返回。返回地址保存到X30寄存器，保存的是父函数的PC+4
* RET: 从子函数返回，通常用X30里保存的返回地址返回。
* ERET：从当前的异常模式返回，通常可以实现模式切换，例如EL1切换到EL0。它会从SPSR恢复PSTATE，从ELR中获取跳转地址，并返回到该地址。

Example:

* 新建一个汇编文件
* 创建一个bl_test的汇编函数，在该汇编函数中使用bl指令来跳转到csel_test汇编函数中
* 在kernel.c文件中，C语言调用该bl_test汇编函数。

分析：

调用顺序应该是main ----->  bl_test -----> csel_test ---> ret， 如果使用bl指令跳转到csel_test中，csel_test返回之后 bl_test拿到的是csel_test返回地址PC，而且**X30寄存器被修改**为csel_test子函数的地址，如果把PC+4返回给main函数，main函数看到的是当前的PC+4，肯定是错误的。这个难点就在于bl跳转子函数之后的返回需要处理好，重置X30寄存器的地址的值。

```assembly
test_csel:

	mov x2, #0x2
	mov x3, #-0x1
	cmp x0, #0
	csel x4, x2, x3, EQ
	add x0, x1, x4
	ret

test_bl:
	mov x8, x30    // 备份main函数call进来x30的lr的值，否则到这里函数回不去
	mov x0, 1
	mov x1, 3
	bl test_csel   // 跳转进入这个函数之后x30寄存器被test_csel子函数冲走了
	mov x30, x8    // 恢复x30的值，此时ret之后可以回到main函数的地址继续执行
	retå
```

## Ref

[^1]: [ARM Cortex-A Series Programmer's Guide for ARMv8-A -Conditional instructions](https://developer.arm.com/documentation/den0024/a/The-A64-instruction-set/Data-processing-instructions/Conditional-instructions?lang=en)