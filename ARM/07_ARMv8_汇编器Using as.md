# 07_ARMv8_汇编器Using as

* 关键字
* 伪指令
* 汇编宏

## 1 关键字

### 1.1 label 和 symbol

代表它所在的地址，也可以当做变量或者符号。

* 全局symbol：`.global xxxxx`
* 局部symbol (label)：在函数内使用，开头以0-99直接的数字为标号名，通常和`b`指令结合使用。
* `f`：指示编译器向前搜索
* `b`：指示编译器向后搜索

```assembly
.global my_sym

my_sym:
		mov x1, 0x80000
		mov x2, 0x200000
		add x3, x1, 32
1:
		ldr x4, [x1], #8
		str x4, [x2], #8
		cmp x1, x3
		b.cc 1b								# 向1 symbol后面的代码执行。
		
		ret
```

### 1.2 伪指令

#### 对齐伪指令.align

声明align后面的汇编必须从下一个能被2^n整除的地址开始分配。ARM64系统中，第一个参数表示2^n大小。

Define: `.align [abs-expr[, abs-expr[, abs-expr]]`

```assembly
.align 3						# 2^3 = 8 字节对齐
.global string1
string1:
		.string "Boot at EL"
```

string1的地址必须是能被8整除的。

#### 数据定义伪指令

* `.byte` 把8位数当做数据插入到汇编当中
* `.hword` 把16位数当成数据插入到汇编当中
* `.long`和`.int`：把32位数当成数据插入到汇编当中
* `.quad`：64位数
* `.float`：浮点数
* `.ascii "string"`：把string当做数据插入到汇编当中，ascii未操作定义字符串需要自行插入‘\0’
* `.asciz "string"`:  不需要手动插入'\0'的string。

#### 执行伪指令

* `.rept`：重复执行伪指令
* `.equ`和`.set`：赋值操作 `.equ symbol, expression`

```assembly
.rept 3
.long 0
.endr
# 等效于
.long 0
.long 0
.long 0
```

```assembly
# 赋值例子
.equ data01, 100
.equ data02, 50

.global main
main:
	ldr x2, =data01
	ldr x3, =data02
```

#### 函数相关伪指令

* `.gloabl`：定义一个全局的符号，可以为变量可以为函数
* `.include`：引用头文件
* `.if, .else, .endif`：控制语句
* `.ifdef symbol`：判断symbol是否定义
* `.ifndef symbol`：未定义
* `.ifc string1, string2`：判断两个字符串是否相等
* `.ifeq expression`: 判断expression的值是否为0
* `.ifeqs string1, string2`： 等同于ifc
* `.ifge expression`：判断值是否≥0 
* `.ifle expression`：判断值是否≤0
* `.ifne expression`：判断值是否不为0

| 运算符           | 说明                                |
| ---------------- | ----------------------------------- |
| expr1 == expr2   | 若 expr1 等于 expr2，则返回“真”     |
| expr1 != expr2   | 若 expr1 不等于 expr2，则返回“真”   |
| expr1 > expr2    | 若 expr1 大于 expr2，则返回"真”     |
| expr1 ≥ expr2    | 若 expr1 大于等于 expr2，则返回“真” |
| expr1 < expr2    | 若 expr1 小于 expr2，则返回“真”     |
| expr1 ≤ expr2    | 若 expr1 小于等于 expr2，则返回“真” |
| !expr1           | 若 expr 为假，则返回“真”            |
| expr1expr2       | 对 expr1 和 expr2 执行逻辑 AND 运算 |
| expr1 \|\| expr2 | 对 1xprl 和 expr2 执行逻辑 OR 运算  |
| expr1 & expr2    | 对 expr1 和 expr2 执行按位 AND 运算 |
| CARR1?           | 若进位标志位置 11则返回“真”         |
| OVERFLOW ?       | 若溢出标志位置 1，则返回“真”        |
| PARITY ?         | 若奇偶标志位置 1，则返回“真”        |
| SIGN ?           | 若符号标志位置 1，则返回“真”        |
| ZERO ?           | 若零标志位置 1，则返回“真”          |

#### 段相关的伪指令

##### .section

`.section name, "flag"`

flag就是ELF文件中的adewxMSGT？可以查看文档[^2],我们在[02_ELF文件结构_浅析内部文件结构](https://github.com/carloscn/blog/issues/6)[^3]中描述了ELF的段的结构，可以参考。

> - `b`
>
>   bss section (uninitialized data)
>
> - `n`
>
>   section is not loaded
>
> - `w`
>
>   writable section
>
> - `d`
>
>   data section
>
> - `r`
>
>   read-only section
>
> - `x`
>
>   executable section
>
> If no flags are specified, the default flags depend upon the section name. If the section name is not recognized, the default will be for the section to be loaded and writable.
>
> If the optional argument to the `.section` directive is not quoted, it is taken as a subsegment number (see section [Sub-Sections](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_node/as_43.html#SEC43)).
>
> For ELF targets, the `.section` directive is used like this:
>
> ```
> .section name[, "flags"[, @type]]
> ```
>
> The optional flags argument is a quoted string which may contain any combintion of the following characters:
>
> - `a`
>
>   section is allocatable
>
> - `w`
>
>   section is writable
>
> - `x`
>
>   section is executable

例子：把符号映射到.idmap.text段，属性为awx

```assembly
.section ".idmap.text", "awx"
```

```assembly
#-----------------------------------
.section .data
.algin 3
.global my_data0
.my_data0:
		.dword 0x00
#----------------------------------- mapping to .data section
.section .text
......
#----------------------------------- mapping to .text section
```

##### .pushsection

`.pushsection`：下面的代码push接到指定的section当中。

##### .popsection

`.popsection`：结束push

两个伪指令需要成对使用，仅仅是把pushsection和popsection的圈出来的代码复制链接到指定的section中，其他代码还在原来的section。

```assembly
.section .text
.global my_add_func
my_add_func:
	...
	ret
	
.pushsection ".idmap.text", "awx"
.global my_sub_func
my_sub_func:
	.....
	ret
.popsection

....
```

这个例子就是把my_sub_func从 .text段里面提取出来然后链接到.idmap.text里面。

#### 实验

【题目】

使用汇编的数据定义伪指令，可以实现表的定义，例如Linux内核使用.quad和.asciz定义了一个kallsyms的表，地址和函数名的对应关系：

* 0x800800 -> func_a
* 0x800860-> func_b
* 0x800880-> func_c

请在汇编里定义一个类似的表，在C语言中根据函数的地址查找表，并且正确打印函数的名称。这个非常常用，我们在调试死机状态的时候可以根据这样的方法把函数的调用名称和栈指针打出来，方便我们调试。

```assembly
.align 3
.global func_addr
func_addr:
	.qual 0x800800
	.qual 0x800860
	.qual 0x800880

.align 3
.global func_str
func_str:
	.asciz "func_a"
	.asciz "func_b"
	.asciz "func_c"

.align 3
.global func_num_syms
func_num_syms:
	.quad 3
```

```C
extern unsigned long func_addr[];
extern char func_str[];
extern unsigned long func_num;
extern int add_f(int a, int b, int c);

static int print_func_name(unsigned long addr)
{
	int i = 0;
	char *p, *str;

	for (i = 0; i < func_num; i++) {
		if (addr == func_addr[i]) {
			goto found;
		}
	}
	uart_send_string("not found func\n");

found:
	p = (char *)&func_str;
	while(1) {
		p++;
		if (*p == '\0') {
			i--;
		}
		if (i == 0) {
			p++;
			str = p;
			uart_send_string(str);
			break;
		}
	}
	return 0;
}
```

## 2 汇编宏

### 2.1 定义

`.macro macname macargs ...`

`.endm`

示例1：

```assembly
.macro add p1 p2
add x0, \p1, \p2
.endm

#可以配置初始值
.macro reserve_str p1=0, p2

#特殊字符（错误使用）
.macro opcode base length
\base.\length
.endm

.macro opcode base length
\base\().\length
.endm
```

在kernel里面有这样的代码：

```assembly
.macro kernel_ventry, el, label, regsize = 64
.algin 7
sub sp, sp, #S_FRAME_SIZE
b el\()\el()_\label
.endm
```

最后结果是b el1_irq

### 2.2 实验

在汇编文件通过如下两个函数：

* long add_1(a, b)
* long add_2(a, b)

然后写一个宏定义

.macro add a, b, label

`#这里调用add_1或者add_2函数，label等于1或者2`

.endm

```assembly
.align 3
add_1:
	add x0, x0, #1
	add x0, x0, x1
	ret

.align 3
add_2:
	add x0, x0, #2
	add x0, x0, x1
	ret

.global add_func
.macro add_func a, b, label
.align 3
	mov x0, \a
	mov x1, \b
	bl add_\()\label
.endm


macro1:
	mov x9, x30
	add_func x0, x1, 1
	mov x30, x9
	ret

macro2:
	mov x9, x30
	add_func x0, x1, 2
	mov x30, x9
	ret

.global add_f
.align 3
add_f:
	mov x9, x30
	cmp x2, 1
	b.eq macro1

	cmp x2, 2
	b.eq macro2

	mov x30, x9
	ret
```

# Ref

[^1]:[ARM64体系架构编程与实现]()
[^2]:[ Using as - GNU Assembler, [`.section name`]](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_node/as_117.html#SEC119)
[^3]: [02_ELF文件结构_浅析内部文件结构](https://github.com/carloscn/blog/issues/6)

