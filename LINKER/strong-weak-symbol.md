# Strong and Weak Symbol Ref

* extern.c 

```C
// extern.c
#include <stdio.h>

// strong symbol
int symbol = 7;

int extern_func(int a, int b)
{
    printf("extern.c : the value a is %d, b is %d\n", a, b);
    return a + b;
}
```

* main.c

```C
#include <stdio.h>

// weak symbol
__attribute__((weak)) int symbol = 1;
__attribute__((weak)) int extern_func(int a, int b);

int main(void)
{
    static int var_1 = 85;
    static int var_2;
    int c = 6;
    int d;

    if (extern_func) {
        extern_func(c, d);
    } else {
        printf("no extern_func impl\n");
    }
    return c;
}
```

### Exp1  weak symbol covered by strong symbol

在extern.c中定义的extern_func，因此extern_func属于强符号，此时main中extern_func会被强制链接到extern.c中的extern_func，实验现象是可以输出真正的实现：

`gcc -c main.c -o main.o`

`gcc -c extern.c -o extern.o`

`gcc main.o extern.o -c main`

```
➜  work ./main 
extern.c : the value a is 6, b is 0
```

### Exp2  weak symbol only

在extern.c中定义的extern_func，因此extern_func属于强符号，此时main中extern_func会被强制链接到extern.c中的extern_func，实验现象是可以输出真正的实现：

`gcc main.c -o main.o`

```
➜  work ./main 
no extern_func impl
```

[ELF for the Arm® 64-bit Architecture (AArch64)](https://developer.arm.com/documentation/ihi0056/latest?_ga=2.56942954.1506853196.1533541889-405231439.1528186050)
