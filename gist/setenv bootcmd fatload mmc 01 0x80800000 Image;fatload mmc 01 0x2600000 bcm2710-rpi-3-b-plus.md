



setenv bootargs "console=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait ip=192.168.31.32::192.168.1.1:255.255.255.0::eth0:off"

setenv bootcmd "fatload mmc 0:1 0x80800000 Image;fatload mmc 0:1 0x83000000 bcm2711-rpi-4-b.dtb;booti 0x80800000 - 0x83000000"

setenv bootcmd "fatload mmc 0:1 0x80800000 Image;booti 0x80800000 - ${fdt_addr}"


setenv bootargs "8250.nr_uarts=1 console=ttyS0,115200 root=/dev/mmcblk1p2 rootfstype=ext4 rootwait rw"


setenv bootargs '8250.nr_uarts=1 console=ttyS0,115200 root=/dev/nfs nfsroot=192.168.31.6:/home/carlos/nfs/rootfs,proto=tcp rw ip=192.168.31.32:192.168.31.6:192.168.31.1:255.255.255.0::eth0:off'



setenv bootcmd "fatload mmc 0:1 ${kernel_addr_r} Image;booti ${kernel_addr_r} - ${fdt_addr}"

setenv bootargs "8250.nr_uarts=1 console=ttyS0,115200 root=/dev/mmcblk0p2 rw rootwait init=/bin/sh"
