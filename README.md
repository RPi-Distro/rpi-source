There is a kernel-headers Raspberry Pi OS package now:
https://www.raspberrypi.com/documentation/computers/linux_kernel.html#kernel-headers

rpi-source is helpful if you use rpi-update kernels or want to build an in-kernel module.

------

rpi-source installs the kernel source used to build rpi-update kernels and the kernel on the Raspberry Pi OS image.
This makes it possible to build loadable kernel modules.
It is not possible to build modules that depend on missing parts that need to be built into the kernel proper (bool in Kconfig).

The script uses sudo internally when self-updating and when making the links */lib/modules/$(uname -r)/{build,source}*

[Examples on how to build various modules](#examples-on-how-to-build-various-modules)

Dependencies
```text
sudo apt install git bc bison flex libssl-dev
```

Install
```text
sudo wget https://raw.githubusercontent.com/RPi-Distro/rpi-source/master/rpi-source -O /usr/local/bin/rpi-source && sudo chmod +x /usr/local/bin/rpi-source && /usr/local/bin/rpi-source -q --tag-update

```
Run
```text
rpi-source
```

### ncurses-devel

This library is needed when running 'make menuconfig'

```text
 *** ncurses-devel is NOT installed. Needed by 'make menuconfig'. On Raspberry Pi OS: sudo apt install libncurses5-dev
```

Install prerequisites
```text
$ sudo apt install libncurses5-dev
```

## Misc

### make prepare
Always run 'make prepare' after changing the kernel config

Show kernel config diff between current and previous config
```text
$ scripts/diffconfig
 ENC28J60 n -> m
+ENC28J60_WRITEVERIFY n
```
Show config diff against the running kernel
```text
$ cd linux
$ zcat /proc/config.gz > .config.running
$ scripts/diffconfig .config.running .config
```

### ~/linux/Module.symvers

Module.symvers contains all the symbols the kernel and modules has exported, and was made during the kernel build process.
This file is used later to find needed symbols when building modules. If it was missing (or empty) during building of a module, you might get a not very helpful error message when loading the module. But dmesg will help:
```text
$ sudo modprobe elo
ERROR: could not insert 'elo': Exec format error
$ dmesg |tail -n 1
[19988.002342] elo: no symbol version for module_layout
```

But if we build a module that exports a symbol, and another module we build needs that symbol, we get warnings.  
The build system can't find that symbol in ~/linux/Module.symvers.  
We need to tell the build system where this symbol is. This can be done with KBUILD_EXTRA_SYMBOLS pointing to the needed Module.symvers in the build directory of the exporting module.  
See [mcp2515a](#mcp2515a) for an example.

* [6.3 Symbols From Another External Module](https://www.kernel.org/doc/Documentation/kbuild/modules.txt)

I had a situation after building several modules that loading the module failed with some format error. It turned out that ~/linux/Module.symvers had changed somehow.  
If that happens, there is a backup file Module.symvers.backup. The timestamps will differ if it has changed.


### Kernel taint
The kernel can be tainted in various ways. One of them is loading an out-of-tree module.
```text
$ cat /proc/sys/kernel/tainted
4096
```
From https://www.kernel.org/doc/Documentation/sysctl/kernel.txt
```text
4096 - An out-of-tree module has been loaded.
```

### Examples on how to build various modules

* [Hello World example](#hello-world-example)
* [spi-config](#spi-config)
* [ENC28J60 SPI Ethernet driver](#enc28j60-spi-ethernet-driver)
* [Microchip MCP251x SPI CAN controller](#microchip-mcp251x-spi-can-controller)
* [mcp2515a](#mcp2515a)
* [PCF8574](#pcf8574)
* [TP-Link TL-WN725N version 2 lwfinger](#tp-link-tl-wn725n-version-2-lwfinger)

When adding a section to this page, please add to the TOC as well.  
Everyone with a Github account can edit this page.

### Hello World example

Classic Hello World example for a loadable kernel module ([LKM](http://en.wikipedia.org/wiki/Loadable_kernel_module)).

```text
mkdir hello && cd hello
```
Makefile
```text
obj-m := hello.o
```
hello.c
```text
#include <linux/module.h>
#include <linux/kernel.h>

int hello_init(void)
{
	pr_alert("Hello World :)\n");
	return 0;
}
void hello_exit(void)
{
	pr_alert("Goodbye World!\n");
}
module_init(hello_init);
module_exit(hello_exit);
```

```text
$ make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
$ sudo insmod hello.ko
$ dmesg | tail -1
[58728.008906] Hello World :)
$ sudo rmmod hello
$ dmesg | tail -1
[58732.440677] Goodbye World!

# Install module (/lib/modules/<ver>/extra/hello.ko)
$ sudo make -C /lib/modules/$(uname -r)/build M=$(pwd) modules_install
$ sudo depmod
$ sudo modprobe hello
```


### spi-config
spi board configuration without having to recompile the kernel  
[README](https://github.com/msperl/spi-config/blob/master/README.md)

```text
$ uname -a
Linux raspberrypi 3.12.18+ #679 PREEMPT Thu May 1 14:40:27 BST 2014 armv6l GNU/Linux

$ git clone https://github.com/msperl/spi-config
$ cd spi-config
$ make
$ sudo make install
$ sudo depmod
```



### ENC28J60 SPI Ethernet driver

Forum post: http://www.raspberrypi.org/phpBB3/viewtopic.php?f=44&t=18397

spi-config can be used to add SPI device

```text
$ uname -a
Linux raspberrypi 3.12.18+ #679 PREEMPT Thu May 1 14:40:27 BST 2014 armv6l GNU/Linux

$ cd linux

# Enable ENC28J60:
# Device Drivers -> Network device support -> Ethernet driver support -> Microchip devices -> <M> ENC28J60 support
$ make menuconfig

# show config diff
$ scripts/diffconfig
 ENC28J60 n -> m
+ENC28J60_WRITEVERIFY n

# prepare for build (always after config change)
$ make prepare

# build
$ make SUBDIRS=drivers/net/ethernet/microchip modules

# install
$ sudo make SUBDIRS=drivers/net/ethernet/microchip modules_install
# this is needed even if the previous command runs depmod (don't know why)
$ sudo depmod

# load
$ sudo modprobe enc28j60
$ lsmod
Module                  Size  Used by
enc28j60               17844  0
```

### Microchip MCP251x SPI CAN controller

spi-config can be used to add SPI device

```text
$ uname -a
Linux raspberrypi 3.12.18+ #679 PREEMPT Thu May 1 14:40:27 BST 2014 armv6l GNU/Linux

$ cd linux

# Networking support -> <M> CAN bus subsystem support
#     CAN Device Drivers -> <M> Microchip MCP251x SPI CAN controllers
$ make menuconfig

$ scripts/diffconfig
 CAN n -> m
+CAN_8DEV_USB n
+CAN_AT91 n
+CAN_BCM m
+CAN_CALC_BITTIMING y
+CAN_CC770 n
+CAN_C_CAN n
+CAN_DEBUG_DEVICES n
+CAN_DEV m
+CAN_EMS_USB n
+CAN_ESD_USB2 n
+CAN_GW m
+CAN_KVASER_USB n
+CAN_LEDS n
+CAN_MCP251X m
+CAN_PEAK_USB n
+CAN_RAW m
+CAN_SJA1000 n
+CAN_SLCAN n
+CAN_SOFTING n
+CAN_VCAN n
+NET_EMATCH_CANID n

# prepare for build (always after config change)
$ make prepare

# build
$ make SUBDIRS=net/can modules
  CC [M]  net/can/bcm.o
  CC [M]  net/can/gw.o
  CC [M]  net/can/raw.o
  CC [M]  net/can/af_can.o
  CC [M]  net/can/proc.o
  LD [M]  net/can/can.o
  LD [M]  net/can/can-raw.o
  LD [M]  net/can/can-bcm.o
  LD [M]  net/can/can-gw.o
  Building modules, stage 2.
  MODPOST 4 modules
  CC      net/can/can-bcm.mod.o
  LD [M]  net/can/can-bcm.ko
  CC      net/can/can-gw.mod.o
  LD [M]  net/can/can-gw.ko
  CC      net/can/can-raw.mod.o
  LD [M]  net/can/can-raw.ko
  CC      net/can/can.mod.o
  LD [M]  net/can/can.ko

$ make SUBDIRS=drivers/net/can modules
  CC [M]  drivers/net/can/dev.o
  LD [M]  drivers/net/can/can-dev.o
  CC [M]  drivers/net/can/mcp251x.o
  Building modules, stage 2.
  MODPOST 2 modules
  CC      drivers/net/can/can-dev.mod.o
  LD [M]  drivers/net/can/can-dev.ko
  CC      drivers/net/can/mcp251x.mod.o
  LD [M]  drivers/net/can/mcp251x.ko

# install
$ sudo make SUBDIRS=net/can modules_install
$ sudo make SUBDIRS=drivers/net/can modules_install
$ sudo depmod

# load
$ sudo modprobe mcp251x
$ dmesg | tail
[16352.509840] CAN device driver interface
$ lsmod
Module                  Size  Used by
mcp251x                 9589  0
can_dev                 9774  1 mcp251x
```

Use spi-config to add device
```text
sudo modprobe spi-config devices=bus=0:cs=0:modalias=mcp2515:speed=10000000:gpioirq=25:pd=20:pdu32-0=16000000:pdu32-4=0x2002
```

Links:
* http://elinux.org/RPi_CANBus


### mcp2515a
New implementation of the mcp251x driver with low latency/low IRQ in mind making use of asynchronous SPI

First install the CAN subsystem modules mentioned in the previous example.

```text
$ uname -a
Linux raspberrypi 3.12.18+ #680 PREEMPT Sat May 3 19:29:46 BST 2014 armv6l GNU/Linux

$ cd
$ git clone https://github.com/msperl/mcp2515a.git
$ cd mcp2515a
```
Remove enc28j60.o from the Makefile

KBUILD_EXTRA_SYMBOLS is needed to make the can_dev symbols available ([6.3 Symbols From Another External Module](https://www.kernel.org/doc/Documentation/kbuild/modules.txt)).
```text
$ make XTRA="KBUILD_EXTRA_SYMBOLS=~/linux/drivers/net/can/Module.symvers"
make -C /lib/modules/3.12.18+/build KBUILD_EXTRA_SYMBOLS=~/linux/drivers/net/can/Module.symvers M=/home/pi/mcp2515a modules
make[1]: Entering directory `/home/pi/linux-b09a27249d61475e4423607f7632a5aa6e7b3a53'
  CC [M]  /home/pi/mcp2515a/mcp2515a.o
/home/pi/mcp2515a/mcp2515a.c: In function âmcp2515a_free_transfersâ:
/home/pi/mcp2515a/mcp2515a.c:463:6: warning: unused variable âiâ [-Wunused-variable]
/home/pi/mcp2515a/mcp2515a.c: At top level:
/home/pi/mcp2515a/mcp2515a.c:72:13: warning: âset_lowâ defined but not used [-Wunused-function]
/home/pi/mcp2515a/mcp2515a.c:79:13: warning: âset_highâ defined but not used [-Wunused-function]
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/pi/mcp2515a/mcp2515a.mod.o
  LD [M]  /home/pi/mcp2515a/mcp2515a.ko
make[1]: Leaving directory `/home/pi/linux-b09a27249d61475e4423607f7632a5aa6e7b3a53'

$ sudo make XTRA="KBUILD_EXTRA_SYMBOLS=~/linux/drivers/net/can/Module.symvers" install
make -C /lib/modules/3.12.18+/build KBUILD_EXTRA_SYMBOLS=~/linux/drivers/net/can/Module.symvers M=/home/pi/mcp2515a modules modules_install
make[1]: Entering directory `/home/pi/linux-b09a27249d61475e4423607f7632a5aa6e7b3a53'
  LD      /home/pi/mcp2515a/built-in.o
  Building modules, stage 2.
  MODPOST 1 modules
  INSTALL /home/pi/mcp2515a/mcp2515a.ko
  DEPMOD  3.12.18+
make[1]: Leaving directory `/home/pi/linux-b09a27249d61475e4423607f7632a5aa6e7b3a53'
sync

$ sudo depmod
$ sudo modprobe mcp2515a
$ lsmod
Module                  Size  Used by
mcp2515a               12523  0
can_dev                 9927  1 mcp2515a

```


### PCF8574

```text
$ uname -a
Linux raspberrypi 3.12.18+ #679 PREEMPT Thu May 1 14:40:27 BST 2014 armv6l GNU/Linux

$ cd linux

# Enable PCF857x:
# Device Drivers -> -*- GPIO Support -> <M> PCF857x, PCA{85,96}7x, and MAX732[89] I2C GPIO expanders
$ make menuconfig
$ make prepare

# build
$ make SUBDIRS=drivers/gpio modules
  CC [M]  drivers/gpio/gpio-pcf857x.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      drivers/gpio/gpio-pcf857x.mod.o
  LD [M]  drivers/gpio/gpio-pcf857x.ko

# install
$ sudo make SUBDIRS=drivers/gpio modules_install
  INSTALL drivers/gpio/gpio-pcf857x.ko
  DEPMOD  3.12.18+
$ sudo depmod

# load driver
$ sudo modprobe i2c-bcm2708
$ sudo modprobe i2c-dev
$ sudo modprobe gpio-pcf857x
$ echo pcf8574 0x39 | sudo tee /sys/bus/i2c/devices/i2c-1/new_device

# Output added by mikroskeem
$ dmesg | tail
[  703.075077] pcf857x 1-0038: probed
[  703.080759] i2c i2c-1: new_device: Instantiated device pcf8574 at 0x38

```


### TP-Link TL-WN725N version 2 lwfinger
Ref: http://tech.enekochan.com/2014/03/08/new-script-to-compile-tp-link-tl-wn725n-version-2-lwfinger-driver-in-raspbian/

```text
$ uname -a
Linux raspberrypi 3.12.21+ #1 PREEMPT Sat Jun 14 13:44:18 CEST 2014 armv6l GNU/Linux

$ git clone https://github.com/lwfinger/rtl8188eu.git
$ cd rtl8188eu

$ make all

$ sudo make install

$ sudo depmod

$ sudo modprobe 8188eu

$ lsmod
Module                  Size  Used by
8188eu                796381  0

```
