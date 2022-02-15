Github地址：[carloscn/uncle-ben-os at car_lab_01 (github.com)](https://github.com/carloscn/uncle-ben-os/tree/car_lab_01)

### ARMv8指令集介绍

* A64指令集只能运行在aarch64
* 所有A64汇编都是32 bits宽的
  * 关注指令的使用、有什么limitation
  * A64能访问的地址数据是64位宽的
* A64支持全部的大写或者小写方式
  * ARM官方大写
  * 应用使用小写
* 寄存器命名
  * Wn表示32bits宽的寄存器
  * Xn表示64bits宽的寄存器
  * WZR表示32位内容全为0的寄存器
  * XZR表示64位内容全为0的寄存器
  * ...

### LDR指令

`LDR Xd, [Xn, $offset]` 

* 【释义】：将Xn寄存器中存储的地址+offset地址偏移存 组成一个新的地址，把这个地址里面存储的值放在Xd寄存器中。[]有取地址内存储的数值的含义。

* 【示例】：

  * S1: 使用MOV指令把0x80000加载到X1寄存器： `MOV x1, 0x80000`  （如果是一个数，而非#0x80000, 则是一个地址）

  * S2: 使用MOV指令把16数值加载到X3寄存器： `MOV x3, 16`

  * S3: 使用LDR指令读取X1地址里面存储的值，存储到X0中:  `LDR x0,[x1] `,  这个不允许->`LDR x2,[0x80000]`

  * S4:使用LDR指令读取X1 + 8地址里面存储的值，存储到X2中：`LDR x2,[x1, #8]`

  * S5:使用LDR指令读取(X1 + X3)地址里面存储的值，存储到X4中： `LDR x4,[x1, x3]`

  * S6: 使用LDR指令读取(X1+(X3<<3))地址里面存储的值，存储到X5中： `LDR x5,[x1,x3,lsl #3]`

* 【注意】：
  * 给的数不加任何标志的视为地址
  * 需要给立即数的场景而非地址的值，使用#
  * []有取地址值的意思
  * LDR lsl扩展指令，只支持1和3
  * `LDR x2,[x1, #8]` x1的值**不会**被更新为0x80008

* 【变基模式】：

  * 前变基模式 pre-index： 先更新偏移地址，后访问地址 （注意有叹号!表示）

    `LDR x6, [x1, #8]!` : 将x1里面的地址增加偏移#8并赋给x1，最后将新的x1寄存器内的地址的值给x6寄存器

  * 后变基模式 post-index： 先访问内存地址，再更新偏移地址

    `LDR x6, [x1], #8` : 将x1寄存器内的地址的值赋给x6寄存器，并将x1地址偏移+8。

### STR指令

从一个寄存器的值吐到内存中，支持立即数和寄存器操作。把Xd的值，存储到[xn|sp]里面。

immediate-post-index: `STR Xd, [Xn|SP], #<simm>` 

immediate-pre-index: `STR Xd, [Xn|SP, #<simm>]!`

* 【示例】：

  ```assembly
  ; Example 1:
  MOV x2, 0x400000             ; -> x2 is 0x400000
  LDR x6, =0x1234abce          ; -> x6 is 0x1234abce
  STR x6, [x2, #8]!            ; -> 把x6的值（0x1234abce），存储到0x400008地址的内存里面
  ; What's value of x2? And the value in 0x400000 address?  
  
  ;Example 2:
  MOV x2, 0x500000			 ; -> x2 is 0x500000
  STR x6, [x2], #8			 ; -> 把x6的值（0x1234abce），存储到0x500000里面，并将x2寄存器变为0x500008
  ; What's value of x2? And the value in 0x400000 address?  
  ```


### MOV指令



## GDB-Tips

* 启动GDB和QEMU链接

  * `> gdb-multiarch --tui benos.elf`
  * `gdb> c`
  * `gdb> target remote localhost:1234`

  * `gdb> b ldr_test`   // 设定断点
  * `gdb> c`
  * `gdb> next`  //下一步
  * `gdb> info register`  // 查看所有寄存器
  * `gdb> info x1 x2 x3` // 查看x1/x2/x3寄存器
  * `gdb> x 0x80000`  // 读取内存0x80000值 32位
  * `gdb> x/xg 0x80000` // 读取内存0x80000值64位