# 实习项目

## 树莓派rust-for-linux 6.6.0 驱动

## 准备工作

### 总体思路：

1，基于合适的树莓派内核，合入rust-for-linux的 rust补丁
2，编译rust sample 驱动
3，验证树莓派双串口驱动在QEMU的调试
4，编写单个驱动，Aux mini8025驱动为目标串口，用PL011串口来调试驱动

* 开发环境和版本信息：
  Debian 11
  qemu-system-aarch64  7.2.5
  llvm  11
  llvm-cang 11
  kernel branch： rpi-6.6.y

### 树莓派 for QEMU调查

### qemu-ARM32 支持以下型号

```
qemu-system-arm -M help | grep rasp
raspi0               Raspberry Pi Zero (revision 1.2)
raspi1ap             Raspberry Pi A+ (revision 1.1)
raspi2b              Raspberry Pi 2B (revision 1.1)

```

### qemu-64位支持以下型号

```
qemu-system-aarch64 -M help | grep rasp
raspi0               Raspberry Pi Zero (revision 1.2)
raspi1ap             Raspberry Pi A+ (revision 1.1)
raspi2b              Raspberry Pi 2B (revision 1.1)
raspi3ap             Raspberry Pi 3A+ (revision 1.0)
raspi3b              Raspberry Pi 3B (revision 1.2)

```

#### QEMU provides models of the following Raspberry Pi boards:

**`raspi0 and raspi1ap :`**ARM1176JZF-S core, 512 MiB of RAM

**`raspi2b :`**Cortex-A7 (4 cores), 1 GiB of RAM

**`raspi3ap:`**Cortex-A53 (4 cores), 512 MiB of RAM

**`raspi3b`**`:`Cortex-A53 (4 cores), 1 GiB of RAM

#### Implemented devices

> * ARM1176JZF-S, Cortex-A7 or Cortex-A53 CPU
> * Interrupt controller
> * DMA controller
> * Clock and reset controller (CPRMAN)
> * System Timer
> * GPIO controller
> * Serial ports (BCM2835 AUX - 16550 based - and PL011)
> * Random Number Generator (RNG)
> * Frame Buffer
> * USB host (USBH)
> * GPIO controller
> * SD/MMC host controller
> * SoC thermal sensor
> * USB2 host controller (DWC2 and MPHI)
> * MailBox controller (MBOX)
> * VideoCore firmware (property)

#### Missing devices

> * Peripheral SPI controller (SPI)
> * Analog to Digital Converter (ADC)
> * Pulse Width Modulation (PWM)

Raspberry Pi boards (raspi0, raspi1ap, raspi2b, raspi3ap, raspi3b)：
<https://www.qemu.org/docs/master/system/arm/raspi.html>

### kernel for 树莓派选择

树莓派内核信息：
<https://www.raspberrypi.com/documentation/computers/linux_kernel.html>
linux6.6.0内核支持树莓派罗列

| board                           | Qemu Model       | Arch  | rpi-6.6.y                                                  | kernel原生 | rust-for-linux |
| ------------------------------- | ---------------- | ----- | ---------------------------------------------------------- | -------- | -------------- |
| Raspberry Zero                  | raspi0  raspi1ap | arm32 | bcmrpi_defconfig                                           | NA       | NA             |
| Raspberry Pi 1, Zero and Zero W | raspi0 raspi1ap  | arm64 | ?                                                          | NA       | NA             |
| Raspberry Pi 2                  | raspi2b          | arm32 | KERNEL=kernel7  make bcm2709_defconfig                     | NA       | NA             |
| Raspberry Pi 2                  | raspi2b          | arm64 | ?                                                          | NA       | NA             |
| Raspberry Pi  3, 3+,            | raspi3ap         | arm32 | KERNEL=kernel7  make bcm2709_defconfig                     | NA       | NA             |
| Raspberry Pi 4 and 400          | NA               | arm32 | KERNEL=kernel7l  make bcm2711_defconfig                    | NA       | NA             |
| Raspberry Pi 3, 3+              | raspi3b          | arm64 | KERNEL=kernel8  make  bcm2711_defconfigbcm2837-rpi-3-b.dtb | NA       | NA             |
| Raspberry Pi 5                  | NA               | arm64 | KERNEL=kernel_2712 make bcm2712_defconfig                  | NA       | NA             |

基于以上信息，树莓派的官方内核分支对开发板的支持比较全，而QEMU支持的最新的板型是raspi3ap和raspi3b，实际模拟raspi3ap板子开机crash，没有深追原因，最后选择了raspi3b

## 编译内核

先验证树莓派的qemu启动，确保双串口调试然后将rust-for-linux上的rust补丁打到树莓派6.6上

### 内核下载和修改

raspy pi 最新分支6.6
<https://github.com/raspberrypi/linux> -b brpi-6.6.y

内核配置
make  ARCH=arm64 LLVM=1 O=build KERNEL=kernel8 bcm2711_defconfig
打开双串口调试，修改文件../arch/arm64/configs/bcm2711_defconfig
CONFIG_SERIAL_8250_RUNTIME_UARTS=5，否则8250驱动不能工作

内核编译
make  ARCH=arm64 LLVM=1 O=build KERNEL=kernel8 Image modules dtbs

qemu启动内核
 qemu-system-aarch64 -M raspi3b         -m 1024         -smp 4          -dtb linux-rpi/build/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b.dtb         -kernel linux-rpi/build/arch/arm64/boot/Image         -initrd busybox-1.33.2/initramfs.cpio.gz         -net user,hostfwd=tcp::5022-:22         -serial stdio \\   串口PL011驱动
        \-serial pty \\    串口Mini 8025串口 一般会转接到 类似/dev/pts/8的结点上
        \-display none         -append "root=/dev/ram init=/init rw earlycon=pl011,0x3f201000 console=ttyAMA0"
        \#-append "root=/dev/ram init=/init rw earlycon=bcm2835-aux-uart,0x7e215040"

### 编译树莓派上rust hello world驱动

rust-for-linux官方的最新的是6.6-rc4
<https://github.com/Rust-for-Linux/linux> -b rust-dev

* 检查内核编译环境


```
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
make LLVM=1 rustavailable

```

配置rust支持
General setup
        \---> \[\*] Rust support
发现HAS_RUST为N，需要支持Rust，从Rust-for-Linux/linux 代码仓获取patch

发现树莓派的brpi-6.6.y分支缺少rust相关支持，因此从rust-for-linux官方仓提取6.6-rc4之后的patch 

* 提取patch命令：
  git format-patch -s  8a749fd1a8720d4619c91c8b6e7528c0a355c0aa --stdout > rust6.6-rc4.patch
* 应用patch命令：
  git am ../../linux/rust6.6-rc4.patch


```
应用：rust: sync: add `Arc::{from_raw, into_raw}`
应用：rust: workqueue: add low-level workqueue bindings
应用：rust: workqueue: define built-in queues
应用：rust: workqueue: add helper for defining work_struct fields
应用：rust: workqueue: implement `WorkItemPointer` for pointer types
应用：rust: workqueue: add `try_spawn` helper method
应用：rust: workqueue: add examples
应用：rust: arc: add explicit `drop()` around `Box::from_raw()`
应用：rust: upgrade to Rust 1.72.1
应用：MAINTAINERS: update Rust webpage
应用：MAINTAINERS: add Maintainer Entry Profile field for Rust
应用：rust: kernel: remove `#[allow(clippy::new_ret_no_self)]`
应用：rust: task: remove redundant explicit link
应用：rust: print: use explicit link in documentation
应用：rust: upgrade to Rust 1.73.0
应用：rust: Use awk instead of recent xargs
应用：rust: Use grep -Ev rather than relying on GNU grep
应用：x86: Enable IBT in Rust if enabled in C
应用：rust: add improved version of `ForeignOwnable::borrow_mut`
应用：rust: bindings: rename const binding using sed
应用：rust: Refactor the build target to allow the use of builtin targets
应用：arm64: rust: Enable Rust support for AArch64
应用：rust: macros: improve `#[vtable]` documentation
应用：add hello world
.git/rebase-apply/patch:10693: trailing whitespace.

.git/rebase-apply/patch:10715: trailing whitespace.

.git/rebase-apply/patch:10717: trailing whitespace.

.git/rebase-apply/patch:10725: trailing whitespace.

.git/rebase-apply/patch:10727: trailing whitespace.

warning: 5 行新增了空白字符误用。

```

### 编译文件系统

### QEMU参数记录

QEMU启动参数参照：<https://www.cnblogs.com/liuhailong0112/p/15042376.html>

serial-port - QEMU 不会创建第二个串行端口(Ubuntu x86-64 guest 和主机)
<https://www.coder.work/article/6767932>
【从零到一的Raspberry】树莓派踩坑实录（一）系统安装与简单开发
<https://blog.csdn.net/SuperiorEE/article/details/127524952>
raspi3b板子的启动参数：

```
qemu-system-aarch64 -kernel  linux-rpi/build/arch/arm64/boot/Image  -initrd  busybox-1.33.2/initramfs.cpio.gz -m 1024M -smp 4 -append "root=/dev/ram rw console=ttyAMA0 earlycon=pl011,0x3f201000" -nographic -M raspi3b -dtb linux-rpi/build/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b.dtb

qemu-system-aarch64 -kernel  linux-rpi/build/arch/arm64/boot/Image  -initrd  rpi_for_rust/rpi-ramfs/busybox-1.33.2/initramfs.cpio.gz -m 1024M -smp 4 -append "root=/dev/ram rw console=pl11 earlycon=pl011,0x3f201000 panic=1"  -M raspi3b -dtb linux-rpi/build/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b.dtb -serial stdio -display none

```

raspi3ap板子的启动参数：

```
 qemu-system-aarch64 -kernel  linux-rpi/build/arch/arm64/boot/Image  -initrd  busybox-1.33.2/initramfs.cpio.gz -m 512M -smp 4 -append "root=/dev/ram rw console=ttyAMA0 earlycon=pl011,0x3f201000" -nographic -M raspi3ap -dtb linux-rpi/build/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-a-plus.dt

```

qemu-system-aarch64 -kernel  linux-rpi/build/arch/arm64/boot/Image  -initrd tools/2020-02-13-raspbian-buster-lite.img -m 1024M -smp 4 -append "root=/dev/ram rw console=ttyAMA0 earlycon=pl011,0x3f201000" -nographic -M raspi3b -dtb linux-rpi/build/arch/arm64/boot/dts/broadcom/bcm2837-rpi-3-b.dtb

#### Qemu 驱动raspbian-buster-lite命令

 qemu-system-aarch64 -M raspi3b -append "earlycon=pl011,0x3f201000 rw earlyprintk loglevel=8 console=ttyAMA0,115200 dwc_otg.lpm_enable=0 root=/dev/mmcblk0p2 rootdelay=1" -dtb ./dtbs/bcm2710-rpi-3-b-plus.dtb -sd ../../tools/2020-02-13-raspbian-buster-lite.img -kernel kernels/kernel8.img -m 1G -smp 4 -serial stdio -usb -device usb-mouse -device usb-kbd -display none

* 替换编译内核
  cd qemu-rpi-kernel/native-emulation
  qemu-system-aarch64 -M raspi3b -append "earlycon=pl011,0x3f201000 rw earlyprintk loglevel=8 console=ttyAMA0,115200 dwc_otg.lpm_enable=0 root=/dev/mmcblk0p2 rootdelay=1" -dtb ./dtbs/bcm2710-rpi-3-b-plus.dtb -sd ../../tools/2020-02-13-raspbian-buster-lite.img -kernel  ../../linux-rpi/build/arch/arm64/boot/Image   -m 1G -smp 4 -serial stdio -usb -device usb-mouse -device usb-kbd --display none

##### raspbian-buster-lite系统的默认密码

raspberrypi login: 
user: pi
password: raspberry

## rust串口调试

### 串口原理和资料

BCM52837有两个串口，一个miniUart 8250 auxiliary UART和一个ARM 的PL011 Uart
串口(PL011)在Linux启动运行过程中扮演的角色 <https://www.cnblogs.com/arnoldlu/p/17441174.html>
<https://www.raspberrypi.com/documentation/computers/configuration.html#configuring-uarts>
原始的MiniUart bcm2835aux串口比较简单没有实现DMA

#### bcm2835aux解读

#### PL011 UART串口手册解读

* 外设概述
  BCM2837-ARM-Peripherals.pdf 文档的第13章为 PL011串口相关介绍 

![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700902753765.png?raw=1)

###### 5个中断

* UARTRXINTR 
  触发中断有两种情况：
  1）发送FIFO使能的情况下，发送FIFO的水平位小于等于触发值时触发水平中断，值到其超过触发水平位，或者主动清除了中断
  2）当FIFO中断禁止使能的情况下，如果发送器里有没有数据可发，则会触发发送中断，向FIFO写数据或者清楚中断，则中断位被清除

• UARTTXINTR 
以下情况的一种则会触发中断：
1）接收FIFO使能的情况下，接收FIFO的水平位到达警戒位，接收中断触发，此时主动清除中断或者低于警戒水平位，则中断位清除
2）如果接收FIFO禁用，一旦有数据接收则会触发中断，此时读取数据或清除中断则接收中断会清除
• UARTRTINTR 
• UARTMSINTR,

##### 寄存器解读

DR 数据寄存器
RSRECR 接收状态错误寄存器
FR  FLAG标志寄存器  TX和RX的FIFO满或空的状态
ILPR 
IBRD 比特利率分频寄存器
LCRH  串口线路控制寄存器 FIFO使能，奇偶校验位等
CR 寄存器使能位 发送和接收使能，CTS，RTS使能位回环等
IFLS 接收和发送警告水位设置
IMSC 中断位屏蔽和置位寄存器
RIS 中断状态寄存器
MIS 中断位屏蔽寄存器
ICR 中断清楚寄存器
DMACR DMA使能寄存器
ITCR 测试寄存器
ITIP 测试寄存器
ITOP 测试寄存器
TDR  数据测试寄存器

#### 驱动编写

PL011驱动参考：
<http://opendocs.containerpi.com/comprehensive-rust/en/bare-metal/aps/uart.html>
<https://google.github.io/comprehensive-rust/bare-metal/aps/uart.html>

<https://google.github.io/comprehensive-rust/zh-CN/bare-metal/aps/uart.html> 

经过对比，整体思路是参考drivers/tty/serial/8250/8250_bcm2835aux.c 原生驱动，进行接口改造实现对应的rust驱动

从fujita的驱动里一致platform相关的trait

```
  GEN     Makefile
  CALL    ../scripts/checksyscalls.sh
  RUSTC L rust/kernel.o
error[E0015]: cannot call non-const fn `<T as RawDeviceId>::to_rawid` in constant functions
   --> ../rust/kernel/driver.rs:174:35
    |
174 |             array.ids[i] = ids[i].to_rawid(offset);
    |                                   ^^^^^^^^^^^^^^^^
    |
    = note: calls in constant functions are limited to constant functions, tuple structs and tuple variants

error: aborting due to previous error

For more information about this error, try `rustc --explain E0015`.
make[4]: *** [../rust/Makefile:466：rust/kernel.o] 错误 1

```

常量函数用不允许调用非常量函数， 关键是~const这个语法还不完善，

Tracking issue for RFC 2632, impl const Trait for Ty and ~const (tilde const) syntax #67792

<https://github.com/rust-lang/rust/issues/67792> 

Tracking Issue for removing impl const and ~const in the standard library #110395

<https://github.com/rust-lang/rust/issues/110395> 

## 总结汇报

* 自我介绍和开发进展

**我叫杨凯，来自西安，现在处于在职状态，职于TCL的通力电子                    工作超过10年，**

我的实习项目是选择的实习项目是rust for linux  树莓派上串口驱动，

10年关注 陈渝老师的公开课  当时还没有一些实际参与的机会

**很荣幸和大家分享我的这次实习感觉和总结**

**因为有这么多优秀的同学和老师，很惭愧实习项目**

收获：

rust 驱动的框架和binging机制，完成 E1000的网卡框架，理解了rust for linux的驱动过程

不足：

树莓派驱动最终还是没有打通

四周 跟着萧老师 陈乐  

在这里我遇到了很多更加优秀

的学生，在他们身上学习了很多

是我的串口调试还没有打通，代码编译存在问题，因此我只能分享下我的思路和具体情况

* 开发思路

**先打通树莓派在Qemu的启动，从树莓派官网下载内核，然后根据rust-for-linux提取补丁应用的树莓派内核源码上，配置树莓派内核双串口**

**QEMU启动的时候一个串口驱动映射到标准输出，一个映射到宿主机的虚拟串口上，一个串口用于调试，一个串口用于编写rust串口驱动**

**我移植的串口驱动是8025的驱动，这个对应的原始C语言驱动代码为250行，调用了platform_driver，tty和serial 驱动接口，由于rust 驱动接口还没有封装这个启动，我现在目前卡在platform rust接口注册上面，尝试移植还未成功**

* 开发感想

**如朱老师的所说  实习的感受：朱义  其实这才是真实的代码世界，之前教学中，老师给我早就铺平了道理，啥坑都已经踩的七七八八的只是因为被安排的明明白白了**

通过这次学习对rust-for-linux驱动过程有了进一步的理解，但认识到我自己对rust编程基础还不扎实，

**同时非常感谢老师们提供这样的一个学习平台，希望后面和大家一起继续共同后学习和进步**

## 树莓派rust-for-linux 6.6.0 驱动


