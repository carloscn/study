# 09_ARMv8_内嵌汇编（内联汇编）Inline assembly

内联汇编并非ARCH64专门的使用方法，而是GNU编译器通用的做法。目的有二，其一，对于时间敏感的函数使用内联汇编减少执行开销；其二，C语言无法访问架构级的特殊指令，比如内存屏障功能。

## 1. 基本用法

### 1.1 基础内联汇编格式

Define:`asm ("asm instruction")`

【示例】：

`asm(icicalluis)` 调用一条高速缓存维护指令。

### 1.2 扩展内联汇编代码

* Define:`asm qualifier-asm (AssemblerInstruction)`

* Instruction: 
  * 格式：指令部分：输出部分：输入部分：损坏部分
  * 编译器不会解析，按照字符串处理
* 限定词：`volatile`,`inline`
  * volatile：无特殊情况下的限定词
  * inline：asm的语句视为代码最小可能性

【示例1】：

```c
static inline unsigned long array_index_mask_nospec(unsigned long idx, unsigned long sz)
{
	unsigned long mask;
  asm volatile(
  "			cmp			%1, %2\n"								// 指令部第一行
  "     sbc     %0, xzr, xzr\n"         // 指令部第二行，使用\n换行
  : "=r" (mask)                         // 输出部分，指定只写属性的变量mask(%0)
  : "r" (idx), "Ir" (sz)                // 输入部分，执行只读部分的变量idx(%1), sz(%2)
  : "cc");                              // 损坏部分，跳出汇编
  csdb();
  return mask;
}
```

【示例2】：

```C
static inline unsigned long arch_local_irq_save(void)
{
  unsigned long flags;
  asm volatile(
  "			mrs		%0, daif\n"
  "     msr   daifset, #2\n"
  : "=r" (flag)
  :
  : "memory"
  );
}
```

#### 1.2.1 修饰符[^1]

| C    | Describe                                                     |
| ---- | ------------------------------------------------------------ |
| =    | Means that this operand is written to by this instruction: the previous value is discarded and replaced by new data. （只能写） |
| +    | Means that this operand is both read and written by the instruction. （读写） |
| &    | Means (in a particular alternative) that this operand is an *earlyclobber* operand, which is written before the instruction is finished using the input operands. Therefore, this operand may not lie in a register that is read by the instruction or as part of any memory address.（输入参数的指令执行完成之后才能写入） |

#### 1.2.2 约束符[^2]

| C    | Describe                                                     |
| ---- | ------------------------------------------------------------ |
| p    | An operand that is a valid memory address is allowed. This is for “load address” and “push address” instructions. （内存地址） |
| m    | A memory operand is allowed, with any kind of address that the machine supports in general. Note that the letter used for the general memory constraint can be re-defined by a back end using the `TARGET_MEM_CONSTRAINT` macro.（内存变量） |
| o    | A memory operand is allowed, but only if the address is *offsettable*. This means that adding a small integer (actually, the width in bytes of the operand, as determined by its machine mode) may be added to the address and the result is also a valid memory address.（内存地址，基地址寻址） |
| r    | A register operand is allowed provided that it is in a general register. 通用寄存器 |
| i    | An immediate integer operand (one with constant value) is allowed. This includes symbolic constants whose values will be known only at assembly time or later. 立即数 |
| V    | A memory operand that is not offsettable. In other words, anything that would fit the ‘m’ constraint but not the ‘o’ constraint. 内存变量，不允许偏移的内存操作数 |
| n    | An immediate integer operand with a known numeric value is allowed. Many systems cannot support assembly-time constants for operands less than a word wide. Constraints for these operands should use ‘n’ rather than ‘i’.离结束 |

除了通用的约束符之外，还有AARCH64特有的约束符[^3]

| C    | Describe                                                     |
| ---- | ------------------------------------------------------------ |
| k    | The stack pointer register (`SP`)                            |
| w    | Floating point register, Advanced SIMD vector register or SVE vector register |
| x    | Like `w`, but restricted to registers 0 to 15 inclusive.     |
| y    | Like `w`, but restricted to registers 0 to 7 inclusive.      |
| Upl  | One of the low eight SVE predicate registers (`P0` to `P7`)  |
| Upa  | Any of the SVE predicate registers (`P0` to `P15`)           |
| I    | Integer constant that is valid as an immediate operand in an `ADD` instruction |
| J    | Integer constant that is valid as an immediate operand in a `SUB` instruction (once negated) |
| K    | Integer constant that can be used with a 32-bit logical instruction |
| L    | Integer constant that can be used with a 32-bit logical instruction |
| M    | Integer constant that is valid as an immediate operand in a 32-bit `MOV` pseudo instruction. The `MOV` may be assembled to one of several different machine instructions depending on the value |
| N    | Integer constant that is valid as an immediate operand in a 64-bit `MOV` pseudo instruction |
| S    | An absolute symbolic address or a label reference            |
| Y    | Floating point constant zero                                 |
| Z    | Integer constant zero                                        |
| Ush  | The high part (bits 12 and upwards) of the pc-relative address of a symbol within 4GB of the instruction |
| Q    | A memory address which uses a single base register with no offset |
| Ump  | A memory address suitable for a load/store pair instruction in SI, DI, SF and DF modes |

【示例3】：分析atomic_add函数内嵌汇编

```C
void my_atomic_add(unsigned long val, void *p)
{
  unsigned long tmp;
  int result;
  asm volatile (
  "		1: 	ldxr %0, [%2]\n"
  "       add %0, %0, %3\n"
  "       stxr %w1, %0, [%2]\n"
  "       cbnz %w1, 1b\m"
  : "+r" (tmp), "+r" (result), "+Q" (*(unsigned long *)p)
  : "r" (val)
  : "cc", "memory"
  );
}
```

内联汇编还支持助记符：

```C
int add(int i, int j) {
  int ret = 0;
  asm volatile (
  "	add %w[result], %w[input_i], %w[input_j]\n"
  : [result] "=r" (res)
  : [input_i] "r" (i), [input_j] "r" (j)
  :
  );
}
```

实验：实现memcpy，使用内联汇编的方式。

```C
void test_memcpy(void)
{
    unsigned long src_addr = 0x80000, dest_addr = 0x200000;
    unsigned long sz = 32;

    asm volatile (
    "       mov x6, %x[_src_addr]\n"
    "       mov x7, %x[_dest_addr]\n"
    "       add x8, x6, %x[_sz]\n"
    "   1:  ldr x9, [x6], #8\n"
    "       str x9, [x7], #8\n"
    "       cmp x6, x8\n"
    "       b.cc 1b\n"
    :
    : [_src_addr] "r" (src_addr), [_dest_addr] "r" (dest_addr), [_sz] "r" (sz)
    : "cc", "memory"
    );
}
```

以下需要注意：

* GDB不能单步调试内联汇编。建议使用纯汇编单独编写调试之后移植到C语言内部。
* 内联汇编的参数属性是易错点。
* 输出部和输入部的修饰符不能用错，否则程序会跑飞。比如参数寄存器x0，指定在输入部，汇编内部对参数做了修改比如用了add指令，那么就跑飞了。

再做一个例子memset

```c
void test_memset(void)
{
    unsigned long addr = 0x80000;
    unsigned long sz = 16;
    unsigned long i = 0;

    asm volatile (
    "       mov x4, #0\n"
    "   1:  stp %x[_count], %x[_count], [%x[_addr]], #16\n"
    "       add %x[_sz], %x[_sz], #16\n"
    "       cmp %x[_sz], %x[_addr]\n"
    "       bne 1b\n"
    : [_addr] "+r" (addr), [_sz] "+r" (sz), [_count] "+r" (i)
    :
    : "memory"
    );
}
```

## 2. 宏函数

内联汇编也可以和C语言的宏联系到一起，使用##字符串拼接的方式，在内联汇编里面使用""双引号来引用宏参数。

```C
#define MY_OPS(ops, asm_ops)                                        \
static inline void my_asm_##ops(unsigned long mask, void *p) {      \
    unsigned long tmp;                                              \
    asm volatile (                                                  \
        "       ldr %1, [%0]\n"                                     \
        "       "#asm_ops" %1, %1, %2\n"                            \
        "       str %1, [%0]\n"                                     \
        : "+r" (p), "+r" (tmp)                                      \
        : "r" (mask)                                                \
        : "memory"                                                  \
    );                                                              \
}

MY_OPS(or, orr)
MY_OPS(and, and)
MY_OPS(andnot, bic)
```

最后宏展开为几个函数：

* my_asm_or
* my_asm_and
* my_asm_andnot

这三个函数只是在内联汇编里面 "#asm_ops"位置不同。

## Ref

[^1]:[GCC - 6.47.3.3 Constraint Modifier Characters ](https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/Modifiers.html#Modifiers)
[^2]:[GCC - 6.47.3.1 Simple Constraints](https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/Simple-Constraints.html#Simple-Constraints)
[^3]:[GCC - 6.47.3.4 Constraints for Particular Machines](https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/Machine-Constraints.html#Machine-Constraints)