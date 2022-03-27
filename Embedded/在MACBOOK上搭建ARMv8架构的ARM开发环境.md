# 在MACBOOK上搭建ARMv8架构的ARM开发环境

本文适用针对于ARMv8-A64指令集的编译器环境搭建。 关于arm的交叉编译环境可以总结以下几点：

* aarch64-linux-gnu： ARMv8的64位架构编译，指令集A64，Linux应用编译。
* aarch64-linux-gnu_ilp32: ARMv8的32位架构编译，指令集A32/T32，Linux应用编译。
* aarch64-none-elf: ARMv8， 64位的裸机编译，指令集A64。
* arm-linux-gnueabi: ARMv7全指令集， ARMv8 aarch32（A32/T32）指令集，Linux应用。
* arm-none-eabi: ARMv7全指令集， ARMv8 aarch32（A32/T32）指令集，裸机应用。

| multiarch name             | syscall ABI | instruction set | endian­ness | word size | description            | spec documents                                               |
| -------------------------- | ----------- | --------------- | ----------- | --------- | ---------------------- | ------------------------------------------------------------ |
| aarch64-linux-gnu          | linux       | ARMv8           | little      | 64        | aarch64 Linux Platform | AAPCS64 (ARM IHI 005A)[1](https://wiki.debian.org/Multiarch/Tuples#fnref-c7e3f48bab06503cfad11314b1aea053fd6f075f) ELF for the ARM 64-bit Architecture[2](https://wiki.debian.org/Multiarch/Tuples#fnref-8063f3fadbf6d71cb0805d7124d5606eda4f3ce0) |
| aarch64_be-linux-gnu       | linux       | ARMv8           | big         | 64        | aarch64 Linux Platform | AAPCS64 (ARM IHI 005A)[1](https://wiki.debian.org/Multiarch/Tuples#fnref-c7e3f48bab06503cfad11314b1aea053fd6f075f) ELF for the ARM 64-bit Architecture[2](https://wiki.debian.org/Multiarch/Tuples#fnref-8063f3fadbf6d71cb0805d7124d5606eda4f3ce0) |
| aarch64-linux-gnu_ilp32    | linux       | ARMv8           | little      | 32        | aarch64 Linux Platform |                                                              |
| aarch64_be-linux-gnu_ilp32 | linux       | ARMv8           | big         | 32        | aarch64 Linux Platform |                                                              |
| arm-linux-gnu              | linux       | ARMv7/ARMv8     | little      | 32        | Old ARM ABI            | APCS (ARM DUI 0041 chapter 9)[4](https://wiki.debian.org/Multiarch/Tuples#fnref-f460c72c3b0aef74f365b695986ecda40eaf37ec) |
| arm-linux-gnueabi          | linux       | ARMv7/ARMv8     | little      | 32        | ARM EABI, soft-float   | AAPCS (ARM IHI 0042D)[5](https://wiki.debian.org/Multiarch/Tuples#fnref-1afc7a5c6e3b1f4f78fe36071dd1eebc87925aed) ARM GNU/Linux ABI Supplement[6](https://wiki.debian.org/Multiarch/Tuples#fnref-cd2c0ea0679f5d05c6d3f9ffd9cc6ef49f7ea0a2) |
| arm-linux-gnueabihf        | linux       | ARMv7/ARMv8     | little      | 32        | ARM EABI, hard-float   | AAPCS (ARM IHI 0042D)[5](https://wiki.debian.org/Multiarch/Tuples#fnref-1afc7a5c6e3b1f4f78fe36071dd1eebc87925aed) and XXXXX (TBD) |
| armeb-linux-gnueabi        | linux       | ARMv7/ARMv8     | big         | 32        | ARM EABI, soft-float   | AAPCS (ARM IHI 0042D)[5](https://wiki.debian.org/Multiarch/Tuples#fnref-1afc7a5c6e3b1f4f78fe36071dd1eebc87925aed) ARM GNU/Linux ABI Supplement[6](https://wiki.debian.org/Multiarch/Tuples#fnref-cd2c0ea0679f5d05c6d3f9ffd9cc6ef49f7ea0a2) |
| armeb-linux-gnueabihf      | linux       | ARMv7/ARMv8     | big         | 32        | ARM EABI, hard-float   | AAPCS (ARM IHI 0042D)[5](https://wiki.debian.org/Multiarch/Tuples#fnref-1afc7a5c6e3b1f4f78fe36071dd1eebc87925aed) and XXXXX (TBD) |
| armv8l-linux-gnueabihf     | linux       | ARMv8           | little      | 32        | ARMv8 EABI, hard-float |                                                              |
| arm-eabi                   | Bare-Metal  | ARMv7/ARMv8     | little      | 32        | ARM EABI, soft-float   |                                                              |
| armeb-eabi                 | Bare-Metal  | ARMv7/ARMv8     | big         | 32        | ARM EABI, soft-float   |                                                              |
| aarch64-elf                | Bare-Metal  | ARMv8           | little      | 64        | ARMv8 EABI, hard-float |                                                              |
| aarch64_be-elf             | Bare-Metal  | ARMv8           | big         | 64        | ARMv8 EABI, hard-float |                                                              |

## 1.1 交叉编译环境

### 1.1.1 A32/T32

```bash
$ brew tap PX4/homebrew-px4
$ brew update
$ brew install gcc-arm-none-eabi-83`
$ arm-none-eabi-gcc --version
arm-none-eabi-gcc (GNU  Tools for Arm Embedded Processors 8-2019-q3-update) 8.3.1 20190703 (release) [gcc-8-branch revision 273027]
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

### 1.1.2 A64

下载地址: https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/downloads

## 1.2 jlink

`brew install openocd`

## 1.3 qemu

自己编译带树莓派4b的qemu

`$ git clone git@github.com:0xMirasio/qemu-patch-raspberry4.git --depth=1`

`$ ./configure --target-list=aarch64-softmmu`

`$ make -j8`

`$ make install`

`$ qemu-system-aarch64 -machine raspi3b -nographic -kernel xxxx.bin -S -s`

不需要树莓派4b支持的可以直接：

`$ brew install qemu`

## 1.4 gdb

`$ aarch64-none-elf-gdb --tui xxxx.elf -x xxxx.sh`



## REF

https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads

https://www.cnblogs.com/arnoldlu/p/14243491.html

https://mynewt.apache.org/v1_5_0/get_started/native_install/cross_tools.html#installing-the-arm-toolchain-for-mac-os-x