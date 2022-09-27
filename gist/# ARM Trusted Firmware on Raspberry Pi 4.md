# ARM Trusted Firmware on Raspberry Pi 4

###### tags: `rpi4` `arm-trusted-firmware` `raspberry pi 4` `security` `ARM TrustZone` `TEE`

>   Security on Raspberry PI 4 model B



-   ARM Trusted Firmware on Raspberry Pi 4
    -   [Basic](https://hackmd.io/@gagachang/atf-on-rpi4#Basic)
    -   [Target Environment](https://hackmd.io/@gagachang/atf-on-rpi4#Target-Environment)
    -   [Build Environment](https://hackmd.io/@gagachang/atf-on-rpi4#Build-Environment)
    -   [Build ARM Trusted-Firmware](https://hackmd.io/@gagachang/atf-on-rpi4#Build-ARM-Trusted-Firmware)
    -   [Set up SD Card with 64-bit Kernel](https://hackmd.io/@gagachang/atf-on-rpi4#Set-up-SD-Card-with-64-bit-Kernel)
    -   [Update rpi-eeprom](https://hackmd.io/@gagachang/atf-on-rpi4#Update-rpi-eeprom)
    -   [Set up ARM Trusted-Firmware](https://hackmd.io/@gagachang/atf-on-rpi4#Set-up-ARM-Trusted-Firmware)
    -   [Boot](https://hackmd.io/@gagachang/atf-on-rpi4#Boot)
    -   [Reference](https://hackmd.io/@gagachang/atf-on-rpi4#Reference)



## Basic

This article is just a note to show how to run ARM trusted-firmware with 64-bit linux kernel, on raspberry pi 4 model B.

Base on [documentation](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/rpi4.rst) of ARM Trusted-Firmware, latest ATF only supports below boot sequence on raspberry pi 4, obviously without secure OS support such as OP-TEE.

```
Power UpVideo Corerpi-eepromloader.binstart.elfATFconfig.txtKernel
```

## Target Environment

-   Rapsberry Pi 4 model B (2018)
-   4 GB RAM
-   32 GB SD card

## Build Environment

-   Ubuntu 20.04 (x86_64) on Virtual Box

## Build ARM Trusted-Firmware

1.  Get ATF source code, version 2.6

    ```
     $ git clone https://github.com/ARM-software/arm-trusted-firmware.git -b v2.6
     $ cd arm-trusted-firmware
    ```

2.  Cross compile

    ```
     $ CROSS_COMPILE=aarch64-linux-gnu- make PLAT=rpi4 DEBUG=1 LOG_LEVEL=50
    ```

3.  Output

    ```
     $ arm-trusted-firmware/build/rpi4/debug/bl31.bin
    ```

## Set up SD Card with 64-bit Kernel

Due to the description in ATF, I select 64-bit linux kernel to test it.

```
You should have AArch64 code in the file loaded as the "kernel",
as BL31 will drop into AArch64/EL2 to the respective load address.
arm64 Linux kernels are known to work this way.
```

https://www.raspberrypi.com/software/

Raspberry pi has officially supported 64-bit kernel, therefore it is not difficult to set up the basic environment. Thanks to the Raspberry Pi Foundation.

According to their official website, we can utilize **Raspberry Pi Imager** to set up 64-bit linux kernel environment on rpi 4.

-   Insert your SD card into the computer
-   Select **Raspberry PI OS (64-bit)** and start

![img](https://raw.githubusercontent.com/carloscn/images/main/typorasGDPTHA.png)

## Update rpi-eeprom

Boot your Pi and connect the network
It is recommended to update the latest **rpi-eeprom** to avoid some potential boot errors.

```
    $ sudo apt update
    $ sudo apt full-upgrade
    $ sudo reboot
```

Log in and check rpi-eeprom version

```
pi@raspberrypi:~$ vcgencmd bootloader_version
2022/01/25 14:30:41
version 6efe41bd9d1e5546fa3715e72e1775b7bd813237 (release)
timestamp 1643121041
update-time 1644737582
capabilities 0x0000007f
```

## Set up ARM Trusted-Firmware

1.  Mount the SD card onto your computer, we can see the boot partition in the SD card.
2.  Copy the **bl31.bin** we just built into this boot partition.
3.  Add new contents into the **config.txt**:

```
    enable_uart=1
    enable_gic=1
    armstub=bl31.bin
```

## Boot

Connect the debug port and power up your Pi with SD card.
We can see ATF boot log and then kernel log:

```
VERBOSE: rpi4: Preparing to boot 64-bit Linux kernel
VERBOSE: Trusted SRAM seen by this BL image: 0x1000 - 0x19000
VERBOSE: Code region: 0x1000 - 0xc000
VERBOSE: Read-only data region: 0xc000 - 0xf000
VERBOSE: Coherent region: 0x18000 - 0x19000
mmap:
 VA:0x0  PA:0x0  size:0x1000  attr:0x9  granularity:0x40000000
 VA:0x1000  PA:0x1000  size:0xb000  attr:0x2  granularity:0x40000000
 VA:0xc000  PA:0xc000  size:0x3000  attr:0x42  granularity:0x40000000
 VA:0x18000  PA:0x18000  size:0x1000  attr:0x8  granularity:0x40000000
 VA:0x1000  PA:0x1000  size:0x18000  attr:0xa  granularity:0x40000000
 VA:0x2ee00000  PA:0x2ee00000  size:0x400000  attr:0x1a  granularity:0x40000000
 VA:0xfc000000  PA:0xfc000000  size:0x4000000  attr:0x8  granularity:0x40000000

VERBOSE: Translation tables state:
VERBOSE:   Xlat regime:     EL3
VERBOSE:   M allowed PA:  0xffffffff
VERBOSE:   Max allowed VA:  0xffffffff
VERBOSE:   Max mapped PA:   0xffffffff
VERBOSE:   Max mapped VA:   0xffffffff
VERBOSE:   Initial lookup level: 1
VERBOSE:   Entries @initial lookup level: 4
VERBOSE:   Used 3 sub-tables out of 4 (spare: 1)
  [LV1] VA:0x0 size:0x40000000
    [LV2] VA:0x0 size:0x200000
      [LV3] VA:0x0 PA:0x0 size:0x1000 NC-RW-XN-S
      [LV3] VA:0x1000 PA:0x1000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x2000 PA:0x2000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x3000 PA:0x3000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x4000 PA:0x4000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x5000 PA:0x5000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x6000 PA:0x6000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x7000 PA:0x7000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x8000 PA:0x8000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0x9000 PA:0x9000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0xa000 PA:0xa000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0xb000 PA:0xb000 size:0x1000 MEM-RO-EXEC-S
      [LV3] VA:0xc000 PA:0xc000 size:0x1000 MEM-RO-XN-S
      [LV3] VA:0xd000 PA:0xd000 size:0x1000 MEM-RO-XN-S
      [LV3] VA:0xe000 PA:0xe000 size:0x1000 MEM-RO-XN-S
      [LV3] VA:0xf000 PA:0xf000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x10000 PA:0x10000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x11000 PA:0x11000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x12000 PA:0x12000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x13000 PA:0x13000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x14000 PA:0x14000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x15000 PA:0x15000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x16000 PA:0x16000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x17000 PA:0x17000 size:0x1000 MEM-RW-XN-S
      [LV3] VA:0x18000 PA:0x18000 size:0x1000 DEV-RW-XN-S
      [LV3] VA:0x19000 size:0x1000
      [LV3] (486 invalid descriptors omitted)
    [LV2] VA:0x200000 size:0x200000
    [LV2] (373 invalid descriptors omitted)
    [LV2] VA:0x2ee00000 PA:0x2ee00000 size:0x200000 MEM-RW-XN-NS
    [LV2] VA:0x2f000000 PA:0x2f000000 size:0x200000 MEM-RW-XN-NS
    [LV2] VA:0x2f200000 size:0x200000
    [LV2] (134 invalid descriptors omitted)
  [LV1] VA:0x40000000 size:0x40000000
  [LV1] (1 invalid descriptors omitted)
  [LV1] VA:0xc0000000 size:0x40000000
    [LV2] VA:0xc0000000 size:0x200000
    [LV2] (479 invalid descriptors omitted)
    [LV2] VA:0xfc000000 PA:0xfc000000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfc200000 PA:0xfc200000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfc400000 PA:0xfc400000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfc600000 PA:0xfc600000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfc800000 PA:0xfc800000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfca00000 PA:0xfca00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfcc00000 PA:0xfcc00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfce00000 PA:0xfce00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfd000000 PA:0xfd000000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfd200000 PA:0xfd200000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfd400000 PA:0xfd400000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfd600000 PA:0xfd600000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfd800000 PA:0xfd800000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfda00000 PA:0xfda00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfdc00000 PA:0xfdc00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfde00000 PA:0xfde00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfe000000 PA:0xfe000000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfe200000 PA:0xfe200000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfe400000 PA:0xfe400000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfe600000 PA:0xfe600000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfe800000 PA:0xfe800000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfea00000 PA:0xfea00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfec00000 PA:0xfec00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xfee00000 PA:0xfee00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xff000000 PA:0xff000000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xff200000 PA:0xff200000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xff400000 PA:0xff400000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xff600000 PA:0xff600000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xff800000 PA:0xff800000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xffa00000 PA:0xffa00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xffc00000 PA:0xffc00000 size:0x200000 DEV-RW-XN-S
    [LV2] VA:0xffe00000 PA:0xffe00000 size:0x200000 DEV-RW-XN-S
NOTICE:  BL31: v2.6(debug):v2.6-dirty
NOTICE:  BL31: Built : 17:10:12, Feb 13 2022
INFO:    Changed device tree to advertise PSCI.
INFO:    ARM GICv2 driver initialized
INFO:    BL31: Initializing runtime services
INFO:    BL31: cortex_a72: CPU workaround for 859971 was applied
WARNING: BL31: cortex_a72: CPU workaround for 1319367 was missing!
INFO:    BL31: cortex_a72: CPU workaround for cve_2017_5715 was applied
INFO:    BL31: cortex_a72: CPU workaround for cve_2018_3639 was applied
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x200000
INFO:    SPSR = 0x3c9
VERBOSE: Argument #0 = 0x2eff3100
VERBOSE: Argument #1 = 0x0
VERBOSE: Argument #2 = 0x0
VERBOSE: Argument #3 = 0x0
VERBOSE: Argument #4 = 0x0
VERBOSE: Argument #5 = 0x0
VERBOSE: Argument #6 = 0x0
VERBOSE: Argument #7 = 0x0
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 5.10.92-v8+ (dom@buildbot) (aarch64-linux-gnu-gcc-8 (Ubuntu/Linaro 8.4.0-3ubuntu1) 8.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #1514 SMP PREEMPT Mon Jan 17 17:39:38 GMT 2022
[    0.000000] random: fast init done
[    0.000000] Machine model: Raspberry Pi 4 Model B Rev 1.2
[    0.000000] efi: UEFI not found.
[    0.000000] Reserved memory: created CMA memory pool at 0x000000001ac00000, size 320 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    0.000000] Zone cache hash table entries: 524288 (order: 10, 4194304 bytes, linear)
[    0.000000] Inode-cache hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] software IO TLB: mapped [mem 0x0000000037400000-0x000000003b400000] (64MB)
[    0.000000] Memory: 3554192K/4050944K available (11136K kernmode
[    0.000000] irq_brcmstb_l2: registered L2 intc (/soc/interrupt-controller@7ef00100, parent irq: 10)
[    0.000000] random: get_random_bytes called from start_kernel+0x3b0/0x570 with crng_init=1
[    0.000000] arch_timer: cp15 timer(s) running at 54.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0xc743ce346, max_idle_ns: 440795203123 ns
[    0.000006] sched_clock: 56 bits at 54MHz, resolution 18ns, wraps every 4398046511102ns
[    0.000249] Console: colour dummy device 80x25
[    0.000836] printk: console [tty1] enabled
```

## Reference

-   Raspberry Pi 4 boot flow
    https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-4-boot-flow
-   ARM Trsuted-Firmware - Raspberry Pi 4
    https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/rpi4.rst
-   樹梅派 Raspberry Pi 更新韌體 EEPROM
    https://cjk.aiao.today/raspberry-pi-update-eeprom/
-   Raspberry Pi 開機流程
    https://hackmd.io/@oscarshiang/raspi_boot
-   Raspberry Pi 4 Arm64 Kernel Cross-Compile
    https://gist.github.com/G-UK/ee7edc4844f14fec12450b2211fc886e
    
    
    
make CROSS_COMPILE=aarch64-linux-gnu- PLAT=rpi4 BOOT_DEVICE=sdmmc BL33=u-boot.bin all fip DEBUG=1 LOG_LEVEL=50
make 