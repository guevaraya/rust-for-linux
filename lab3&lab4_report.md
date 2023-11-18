
## 练习3& 4 编写bindling 和填充checkpoint编写E1000驱动

作业描述：<https://github.com/rcore-os/rust-for-linux/blob/main/exercise3.md> 

参考链接：  https://github.com/yuoo655/e1000-driver.git
# bingding填写
在linux 的rust驱动中需要调用内核API，需要借助bindgen是一个将C 语言的头文件转化为rust源码的工具，才能调用。https://rust-lang.github.io/rust-bindgen/
下面是一个C的头文件

linux/include/r4l_sample.h
```
int echo_for_rust(const unsigned char * str)
{
        printk("[r4l test]:%s\n", str);
        return 0;
}
EXPORT_SYMBOL_GPL(echo_for_rust);
```
linux/rust/helpers.c
````
int rust_helper_echo(const unsigned char * str)
{
        return echo_for_rust(str);
}
EXPORT_SYMBOL_GPL(rust_helper_echo);
````
这样就可以在rust里面通过bingling::echo调用C代码了


# 进入状态

刚开始老师讲解e1000驱动过程，云里雾里，驱动作业无从下手的感觉，前期内核编译和文件系统制作原本很简单，但是各种安装包不兼容问题，并且rust-for-linux的驱动和内核接口变化太快，导致驱动调试，e1000-driver最好还是要基于fujita-linux编译会更好一点，这块我遇到的问题是
先用官网的<https://github.com/Rust-for-Linux/linux的编译并制作好了文件系统，helloworld驱动也验证通过，调试> https\://github.com/yuoo655/e1000-driver.git驱动的时候发现其与<https://github.com/Rust-for-Linux/linux的rust-dev分支匹配起来有很多编译问题，>
随转向 编译<https://github.com/fujita/linux.git，但bindgen要求的版本比较低，结果硬着用新bindgen> 1.62.0 替换1.56.0 编译过去，修改了内核部分代码，花费了三四天才真正开始调驱动，看起其他同学都已经调试完成了，心里还是有点焦急的。但没办法，硬着头皮调试，不能放弃。

# 驱动调试网络原理困惑

由于之前没有接触过网卡驱动，对网卡配置有点手足无措，最后经过调查E1000有fujita的日本人已经调试好的代码参考 <https://github.com/fujita/rust-e1000，有前辈的代码，可以摸着它的代码调试了>

![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700282594632.png?raw=1)

驱动OSI 7层协议，应用层，表示层，会话层，传输层，网络层，链路层和物理层
物理层和链路层对应网络的驱动，应用层，表示层，会话层对应ftp，http，ssh等，而传输层就是UDP，TCP，以及最热门的QUIC协议，而链路层典型的是MAC层，物理层是Phy层，IDH等。

代码经过一系列的折腾和编译，也参考了<https://github.com/lispking/rust-e1000-driver> 这位同学的代码，启动之后结果发现没有probe打印，

![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700234994664.png?raw=1)
经过仔细对吧，我们的开机启动qemu没有加入e1000网卡参数
处理中断的时候没有考虑None的情况，导致panic

```
 -device e1000,netdev=net0,bus=pcie.0 \
        -netdev user,id=net0 \

```

### 网卡数据发送：

HTTP发送网络报过程

套接字缓冲区查询命令ss -ntt![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700284409535.png?raw=1)



![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700282499232.png?raw=1)

看来start_xmit从网络层项链路层发送skb_buffer数据到tx_ring

skb_buffer 数据结构

![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700282810824.png?raw=1)

Ring buffer结构如下：



![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700280752946.png?raw=1)

![](https://308c364b04.oicp.vip/lib/c07fd064-8e60-45b5-81a7-f55c1d5f6217/file/images/auto-upload/image-1700280822212.png?raw=1)

e1000 收发参考原理：<https://blog.csdn.net/u010180372/article/details/119525638>

\[内核源码] Linux 网络数据接收流程（TCP）- NAPI : <https://zhuanlan.zhihu.com/p/452612386> 

# 渐入佳境-驱动debug

* 问题： eth1 up的时候内核崩溃


```
/ # ifconfig eth1 up
[   31.261677] rust_e1000dev: Ethernet E1000 open
[   31.262660] rust_e1000dev: New E1000 device @ 0xffff800008480000
[   31.268236] rust_e1000dev: Allocated vaddr: 0xffff559803193000, paddr: 0x43193000
[   31.271569] rust_e1000dev: Allocated vaddr: 0xffff5598033d0000, paddr: 0x433d0000
[   31.282618] rust_e1000dev: Allocated vaddr: 0xffff559805c80000, paddr: 0x45c80000
[   31.289265] rust_e1000dev: Allocated vaddr: 0xffff559805d00000, paddr: 0x45d00000
[   31.292126] rust_e1000dev: e1000 CTL: 0x140240, Status: 0x80080783
[   31.293155] rust_e1000dev: e1000_init has been completed
[   31.294449] rust_e1000dev: e1000 device is initialized
[   31.302014] rust_e1000dev: handle_irq
[   31.303574] rust_e1000dev: irq::Handler E1000_ICR = 0x4
[   31.307389] rust_e1000dev: NapiPoller poll
[   31.307501] rust_e1000dev: get stats64
[   31.309985] rust_e1000dev: e1000_recv
[   31.309985]
[   31.310547] rust_kernel: panicked at 'called `Option::unwrap()` on a `None` value', /srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux/../e1000_for_linux.rs:86:54
[   31.321218] ------------[ cut here ]------------
[   31.322559] rust_e1000dev: get stats64
[   31.324359] kernel BUG at rust/helpers.c:48!
[   31.326392] Internal error: Oops - BUG: 00000000f2000800 [#1] PREEMPT SMP
[   31.329246] Modules linked in: e1000_for_linux(O)
[   31.335888] CPU: 0 PID: 0 Comm: swapper/0 Tainted: G           O       6.1.0-rc1-g5fc95830739f-dirty #2
[   31.340642] Hardware name: linux,dummy-virt (DT)
[   31.343687] pstate: 60000005 (nZCv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[   31.346427] pc : rust_helper_BUG+0x4/0x8
[   31.350033] lr : rust_begin_unwind+0x5c/0x60
[   31.352394] sp : ffff800008003ca0
[   31.354242] x29: ffff800008003cf0 x28: ffff5598033d0000 x27: ffff800008003de0
[   31.358153] x26: ffff800008003ef0 x25: ffff800008003f00 x24: ffffb7c2d17ca5fd
[   31.361537] x23: ffffb7c31aea4634 x22: 0000000000000001 x21: 0000000000000100
[   31.364296] x20: 0000000000000000 x19: ffff800008003e00 x18: ffffffffffffffff
[   31.366870] x17: 0000000000000041 x16: ffffb7c31adfd5b8 x15: 0000000000000004
[   31.369318] x14: 0000000000000fff x13: ffffb7c31b95b090 x12: 0000000000000003
[   31.372159] x11: 00000000ffffefff x10: c0000000ffffefff x9 : 306c1c7bb7800c00
/ # [   31.374389] x8 : 306c1c7bb7800c00 x7 : 205d373435303133 x6 : 332e31332020205b
[   31.378287] x5 : ffffb7c31bcf7eaf x4 : 0000000000000001 x3 : 0000000000000000
[   31.380756] x2 : 0000000000000000 x1 : ffff800008003a60 x0 : 00000000000000e3
[   31.383739] Call trace:
[   31.384726]  rust_helper_BUG+0x4/0x8
[   31.386219]  _RNvNtCs3yuwAp0waWO_4core9panicking9panic_fmt+0x34/0x38
[   31.389057]  _RNvNtCs3yuwAp0waWO_4core9panicking5panic+0x38/0x3c
[   31.390407]  _RNvXs2_CsejK4Ne9WL8c_15e1000_for_linuxNtB5_6PollerNtNtCsfATHBUcknU9_6kernel3net10NapiPoller4poll+0x560/0x564 [e1000_for_linux]
[   31.396475]  _RNvMs7_NtCsfATHBUcknU9_6kernel3netINtB5_11NapiAdapterNtCsejK4Ne9WL8c_15e1000_for_linux6PollerE13poll_callbackBR_+0x44/0x58 [e1000_for_linux]
[   31.400059]  __napi_poll+0x48/0x1cc
[   31.401478]  net_rx_action+0x134/0x2d4
[   31.402281]  __do_softirq+0xdc/0x25c
[   31.403612]  ____do_softirq+0x10/0x1c
[   31.404441]  call_on_irq_stack+0x2c/0x54
[   31.405413]  do_softirq_own_stack+0x1c/0x28
[   31.406320]  __irq_exit_rcu+0x98/0xec
[   31.407628]  irq_exit_rcu+0x10/0x1c
[   31.408937]  el1_interrupt+0x8c/0xbc
[   31.409942]  el1h_64_irq_handler+0x18/0x24
[   31.411241]  el1h_64_irq+0x64/0x68
[   31.412316]  arch_cpu_idle+0x18/0x28
[   31.413007]  cpuidle_idle_call+0x6c/0x1e4
[   31.413789]  do_idle+0xcc/0xf4
[   31.414408]  cpu_startup_entry+0x24/0x28
[   31.416156]  kernel_init+0x0/0x1a0
[   31.417162]  start_kernel+0x0/0x41c
[   31.418206]  start_kernel+0x328/0x41c
[   31.419322]  __primary_switched+0xbc/0xc4
[   31.421651] Code: a8c17bfd d50323bf d65f03c0 d503233f (d4210000)
[   31.425128] ---[ end trace 0000000000000000 ]---
[   31.427231] Kernel panic - not syncing: Oops - BUG: Fatal exception in interrupt
[   31.428962] SMP: stopping secondary CPUs
[   31.432119] Kernel Offset: 0x37c311e00000 from 0xffff800008000000
[   31.433208] PHYS_OFFSET: 0xffffaa6840000000
[   31.433966] CPU features: 0x20000,2013c080,0000421b
[   31.435508] Memory Limit: none
[   31.437214] ---[ end Kernel panic - not syncing: Oops - BUG: Fatal exception in interrupt ]---

```

边看老师的视频，参考fujita和查找网络资料，终于ping通了

root@openmediavault:~/yangkai/work/rcore_2023a/rust-for_linux# make setup

make -C e1000-driver/src/linux

make\[1]: 进入目录“/srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux”

make ARCH=arm64 LLVM=1 -C ../../../linux-fujita/build M=/srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux/../ modules #rust-analyzer

make\[2]: 进入目录“/srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/linux-fujita/build”

make run

  RUSTC \[M] /srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux/../e1000_for_linux.o

  MODPOST /srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux/../Module.symvers

  LD \[M]  /srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux/../e1000_for_linux.ko

make\[2]: 离开目录“/srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/linux-fujita/build”

make\[1]: 离开目录“/srv/dev-disk-by-uuid-acaca851-c5c0-4f99-8522-91fec8bd0740/users/yangkai/work/rcore_2023a/rust-for_linux/e1000-driver/src/linux”

rm  busybox-1.33.2/\_install/e1000_for_linux.ko

cp  e1000-driver/src/e1000_for_linux.ko   busybox-1.33.2/\_install/

cd busybox-1.33.2/; ./setup.sh; cd -

mkdir: 无法创建目录 “proc”: 文件已存在

mkdir: 无法创建目录 “sys”: 文件已存在

mkdir: 无法创建目录 “dev”: 文件已存在

mkdir: 无法创建目录 “tmp”: 文件已存在

.

./bin

./bin/busybox

./bin/arch

./bin/ash

./bin/base32

./bin/base64

./bin/cat

./bin/chattr

./bin/chgrp

./bin/chmod

./bin/chown

./bin/conspy

./bin/cp

./bin/cpio

./bin/cttyhack

./bin/date

./bin/dd

./bin/df

./bin/dmesg

./bin/dnsdomainname

./bin/dumpkmap

./bin/echo

./bin/ed

./bin/egrep

./bin/false

./bin/fatattr

./bin/fdflush

./bin/fgrep

./bin/fsync

./bin/getopt

./bin/grep

./bin/gunzip

./bin/gzip

./bin/hostname

./bin/hush

./bin/ionice

./bin/iostat

./bin/ipcalc

./bin/kbd_mode

./bin/kill

./bin/link

./bin/linux32

./bin/linux64

./bin/ln

./bin/login

./bin/ls

./bin/lsattr

./bin/lzop

./bin/makemime

./bin/mkdir

./bin/mknod

./bin/mktemp

./bin/more

./bin/mount

./bin/mountpoint

./bin/mpstat

./bin/mt

./bin/mv

./bin/netstat

./bin/nice

./bin/nuke

./bin/pidof

./bin/ping

./bin/ping6

./bin/pipe_progress

./bin/printenv

./bin/ps

./bin/pwd

./bin/reformime

./bin/resume

./bin/rev

./bin/rm

./bin/rmdir

./bin/rpm

./bin/run-parts

./bin/scriptreplay

./bin/sed

./bin/setarch

./bin/setpriv

./bin/setserial

./bin/sh

./bin/sleep

./bin/stat

./bin/stty

./bin/su

./bin/sync

./bin/tar

./bin/touch

./bin/true

./bin/umount

./bin/uname

./bin/usleep

./bin/vi

./bin/watch

./bin/zcat

./bin/rust_helloworld.ko

./linuxrc

./sbin

./sbin/acpid

./sbin/adjtimex

./sbin/arp

./sbin/blkid

./sbin/blockdev

./sbin/bootchartd

./sbin/depmod

./sbin/devmem

./sbin/fbsplash

./sbin/fdisk

./sbin/findfs

./sbin/freeramdisk

./sbin/fsck

./sbin/fsck.minix

./sbin/fstrim

./sbin/getty

./sbin/halt

./sbin/hdparm

./sbin/hwclock

./sbin/ifconfig

./sbin/ifdown

./sbin/ifenslave

./sbin/ifup

./sbin/init

./sbin/insmod

./sbin/ip

./sbin/ipaddr

./sbin/iplink

./sbin/ipneigh

./sbin/iproute

./sbin/iprule

./sbin/iptunnel

./sbin/klogd

./sbin/loadkmap

./sbin/logread

./sbin/losetup

./sbin/lsmod

./sbin/makedevs

./sbin/mdev

./sbin/mkdosfs

./sbin/mke2fs

./sbin/mkfs.ext2

./sbin/mkfs.minix

./sbin/mkfs.vfat

./sbin/mkswap

./sbin/modinfo

./sbin/modprobe

./sbin/nameif

./sbin/pivot_root

./sbin/poweroff

./sbin/raidautorun

./sbin/reboot

./sbin/rmmod

./sbin/route

./sbin/run-init

./sbin/runlevel

./sbin/setconsole

./sbin/slattach

./sbin/start-stop-daemon

./sbin/sulogin

./sbin/swapoff

./sbin/swapon

./sbin/switch_root

./sbin/sysctl

./sbin/syslogd

./sbin/tc

./sbin/tunctl

./sbin/uevent

./sbin/vconfig

./sbin/watchdog

./sbin/zcip

./usr

./usr/bin

./usr/bin/\[

./usr/bin/\[\[

./usr/bin/awk

./usr/bin/basename

./usr/bin/bc

./usr/bin/beep

./usr/bin/blkdiscard

./usr/bin/bunzip2

./usr/bin/bzcat

./usr/bin/bzip2

./usr/bin/cal

./usr/bin/chpst

./usr/bin/chrt

./usr/bin/chvt

./usr/bin/cksum

./usr/bin/clear

./usr/bin/cmp

./usr/bin/comm

./usr/bin/crontab

./usr/bin/cryptpw

./usr/bin/cut

./usr/bin/dc

./usr/bin/deallocvt

./usr/bin/diff

./usr/bin/dirname

./usr/bin/dos2unix

./usr/bin/dpkg

./usr/bin/dpkg-deb

./usr/bin/du

./usr/bin/dumpleases

./usr/bin/eject

./usr/bin/env

./usr/bin/envdir

./usr/bin/envuidgid

./usr/bin/expand

./usr/bin/expr

./usr/bin/factor

./usr/bin/fallocate

./usr/bin/fgconsole

./usr/bin/find

./usr/bin/flock

./usr/bin/fold

./usr/bin/free

./usr/bin/ftpget

./usr/bin/ftpput

./usr/bin/fuser

./usr/bin/groups

./usr/bin/hd

./usr/bin/head

./usr/bin/hexdump

./usr/bin/hexedit

./usr/bin/hostid

./usr/bin/id

./usr/bin/install

./usr/bin/ipcrm

./usr/bin/ipcs

./usr/bin/killall

./usr/bin/last

./usr/bin/less

./usr/bin/logger

./usr/bin/logname

./usr/bin/lpq

./usr/bin/lpr

./usr/bin/lsof

./usr/bin/lspci

./usr/bin/lsscsi

./usr/bin/lsusb

./usr/bin/lzcat

./usr/bin/lzma

./usr/bin/man

./usr/bin/md5sum

./usr/bin/mesg

./usr/bin/microcom

./usr/bin/mkfifo

./usr/bin/mkpasswd

./usr/bin/nc

./usr/bin/nl

./usr/bin/nmeter

./usr/bin/nohup

./usr/bin/nproc

./usr/bin/nsenter

./usr/bin/nslookup

./usr/bin/od

./usr/bin/openvt

./usr/bin/passwd

./usr/bin/paste

./usr/bin/patch

./usr/bin/pgrep

./usr/bin/pkill

./usr/bin/pmap

./usr/bin/printf

./usr/bin/pscan

./usr/bin/pstree

./usr/bin/pwdx

./usr/bin/readlink

./usr/bin/realpath

./usr/bin/renice

./usr/bin/reset

./usr/bin/resize

./usr/bin/rpm2cpio

./usr/bin/runsv

./usr/bin/runsvdir

./usr/bin/rx

./usr/bin/script

./usr/bin/seq

./usr/bin/setfattr

./usr/bin/setkeycodes

./usr/bin/setsid

./usr/bin/setuidgid

./usr/bin/sha1sum

./usr/bin/sha256sum

./usr/bin/sha3sum

./usr/bin/sha512sum

./usr/bin/showkey

./usr/bin/shred

./usr/bin/shuf

./usr/bin/smemcap

./usr/bin/softlimit

./usr/bin/sort

./usr/bin/split

./usr/bin/ssl_client

./usr/bin/strings

./usr/bin/sum

./usr/bin/sv

./usr/bin/svc

./usr/bin/svok

./usr/bin/tac

./usr/bin/tail

./usr/bin/taskset

./usr/bin/tcpsvd

./usr/bin/tee

./usr/bin/telnet

./usr/bin/test

./usr/bin/tftp

./usr/bin/time

./usr/bin/timeout

./usr/bin/top

./usr/bin/tr

./usr/bin/traceroute

./usr/bin/traceroute6

./usr/bin/truncate

./usr/bin/ts

./usr/bin/tty

./usr/bin/ttysize

./usr/bin/udpsvd

./usr/bin/unexpand

./usr/bin/uniq

./usr/bin/unix2dos

./usr/bin/unlink

./usr/bin/unlzma

./usr/bin/unshare

./usr/bin/unxz

./usr/bin/unzip

./usr/bin/uptime

./usr/bin/users

./usr/bin/uudecode

./usr/bin/uuencode

./usr/bin/vlock

./usr/bin/volname

./usr/bin/w

./usr/bin/wall

./usr/bin/wc

./usr/bin/wget

./usr/bin/which

./usr/bin/who

./usr/bin/whoami

./usr/bin/whois

./usr/bin/xargs

./usr/bin/xxd

./usr/bin/xz

./usr/bin/xzcat

./usr/bin/yes

./usr/sbin

./usr/sbin/add-shell

./usr/sbin/addgroup

./usr/sbin/adduser

./usr/sbin/arping

./usr/sbin/brctl

./usr/sbin/chat

./usr/sbin/chpasswd

./usr/sbin/chroot

./usr/sbin/crond

./usr/sbin/delgroup

./usr/sbin/deluser

./usr/sbin/dnsd

./usr/sbin/ether-wake

./usr/sbin/fakeidentd

./usr/sbin/fbset

./usr/sbin/fdformat

./usr/sbin/fsfreeze

./usr/sbin/ftpd

./usr/sbin/httpd

./usr/sbin/i2cdetect

./usr/sbin/i2cdump

./usr/sbin/i2cget

./usr/sbin/i2cset

./usr/sbin/i2ctransfer

./usr/sbin/ifplugd

./usr/sbin/inetd

./usr/sbin/killall5

./usr/sbin/loadfont

./usr/sbin/lpd

./usr/sbin/mim

./usr/sbin/nanddump

./usr/sbin/nandwrite

./usr/sbin/nbd-client

./usr/sbin/nologin

./usr/sbin/ntpd

./usr/sbin/partprobe

./usr/sbin/popmaildir

./usr/sbin/powertop

./usr/sbin/rdate

./usr/sbin/rdev

./usr/sbin/readahead

./usr/sbin/readprofile

./usr/sbin/remove-shell

./usr/sbin/rtcwake

./usr/sbin/sendmail

./usr/sbin/setfont

./usr/sbin/setlogcons

./usr/sbin/svlogd

./usr/sbin/tftpd

./usr/sbin/ubiattach

./usr/sbin/ubidetach

./usr/sbin/ubimkvol

./usr/sbin/ubirename

./usr/sbin/ubirmvol

./usr/sbin/ubirsvol

./usr/sbin/ubiupdatevol

./proc

./sys

./dev

./tmp

./init

./e1000_for_linux.ko

4876 块

/root/yangkai/work/rcore_2023a/rust-for_linux/busybox-1.33.2

/root/yangkai/work/rcore_2023a/rust-for_linux

root@openmediavault:~/yangkai/work/rcore_2023a/rust-for_linux# make run

\#qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel linux/build/arch/arm64/boot/Image -initrd busybox-1.33.2/initramfs.cpio.gz -nographic -append "init=/linuxrc console=ttyAMA0"

qemu-system-aarch64 -M virt \\

\-cpu cortex-a72 \\

\-smp 8 \\

\-m 128M \\

\-device virtio-net-device,netdev=net  \\

\-netdev user,id=net,hostfwd=tcp::2222-:22  \\

\-kernel linux-fujita/build/arch/arm64/boot/Image \\

\-initrd busybox-1.33.2/initramfs.cpio.gz \\

\-nographic \\

\-device e1000,netdev=net0,bus=pcie.0 \\

\-netdev user,id=net0 \\

\-append "init=/linuxrc console=ttyAMA0"

\[    0.000000] Booting Linux on physical CPU 0x0000000000 \[0x410fd083]

\[    0.000000] Linux version 6.1.0-rc1-g5fc95830739f-dirty (root@openmediavault) (Debian clang version 11.0.1-2, LLD 11.0.1) #4 SMP PREEMPT Sat Nov 18 17:07:27 CST 2023

\[    0.000000] random: crng init done

\[    0.000000] Machine model: linux,dummy-virt

\[    0.000000] efi: UEFI not found.

\[    0.000000] NUMA: No NUMA configuration found

\[    0.000000] NUMA: Faking a node at \[mem 0x0000000040000000-0x0000000047ffffff]

\[    0.000000] NUMA: NODE_DATA \[mem 0x47faea00-0x47fb0fff]

\[    0.000000] Zone ranges:

\[    0.000000]   DMA      \[mem 0x0000000040000000-0x0000000047ffffff]

\[    0.000000]   DMA32    empty

\[    0.000000]   Normal   empty

\[    0.000000] Movable zone start for each node

\[    0.000000] Early memory node ranges

\[    0.000000]   node   0: \[mem 0x0000000040000000-0x0000000047ffffff]

\[    0.000000] Initmem setup node 0 \[mem 0x0000000040000000-0x0000000047ffffff]

\[    0.000000] cma: Reserved 32 MiB at 0x0000000045c00000

\[    0.000000] psci: probing for conduit method from DT.

\[    0.000000] psci: PSCIv1.1 detected in firmware.

\[    0.000000] psci: Using standard PSCI v0.2 function IDs

\[    0.000000] psci: Trusted OS migration not required

\[    0.000000] psci: SMC Calling Convention v1.0

\[    0.000000] percpu: Embedded 20 pages/cpu s44840 r8192 d28888 u81920

\[    0.000000] Detected PIPT I-cache on CPU0

\[    0.000000] CPU features: detected: Spectre-v2

\[    0.000000] CPU features: detected: Spectre-v3a

\[    0.000000] CPU features: detected: Spectre-v4

\[    0.000000] CPU features: detected: Spectre-BHB

\[    0.000000] CPU features: kernel page table isolation forced ON by KASLR

\[    0.000000] CPU features: detected: Kernel page table isolation (KPTI)

\[    0.000000] CPU features: detected: ARM erratum 1742098

\[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923

\[    0.000000] alternatives: applying boot alternatives

\[    0.000000] Fallback order for Node 0: 0

\[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32256

\[    0.000000] Policy zone: DMA

\[    0.000000] Kernel command line: init=/linuxrc console=ttyAMA0

\[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes, linear)

\[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)

\[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off

\[    0.000000] Memory: 60668K/131072K available (16512K kernel code, 3726K rwdata, 9232K rodata, 1920K init, 610K bss, 37636K reserved, 32768K cma-reserved)

\[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=8, Nodes=1

\[    0.000000] rcu: Preemptible hierarchical RCU implementation.

\[    0.000000] rcu:     RCU event tracing is enabled.

\[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=8.

\[    0.000000]  Trampoline variant of Tasks RCU enabled.

\[    0.000000]  Tracing variant of Tasks RCU enabled.

\[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.

\[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=8

\[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0

\[    0.000000] Root IRQ handler: gic_handle_irq

\[    0.000000] GICv2m: range\[mem 0x08020000-0x08020fff], SPI\[80:143]

\[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.

\[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).

\[    0.000000] clocksource: arch_sys_counter: mask: 0x1ffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns

\[    0.000256] sched_clock: 57 bits at 63MHz, resolution 16ns, wraps every 4398046511096ns

\[    0.049108] Console: colour dummy device 80x25

\[    0.061200] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)

\[    0.061940] pid_max: default: 32768 minimum: 301

\[    0.066289] LSM: Security Framework initializing

\[    0.081869] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)

\[    0.082129] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)

\[    0.247123] cacheinfo: Unable to detect cache hierarchy for CPU 0

\[    0.278157] cblist_init_generic: Setting adjustable number of callback queues.

\[    0.278734] cblist_init_generic: Setting shift to 3 and lim to 1.

\[    0.280340] cblist_init_generic: Setting shift to 3 and lim to 1.

\[    0.288561] rcu: Hierarchical SRCU implementation.

\[    0.288768] rcu:     Max phase no-delay instances is 1000.

\[    0.320921] EFI services will not be available.

\[    0.329118] smp: Bringing up secondary CPUs ...

\[    0.353431] Detected PIPT I-cache on CPU1

\[    0.358678] cacheinfo: Unable to detect cache hierarchy for CPU 1

\[    0.361510] CPU1: Booted secondary processor 0x0000000001 \[0x410fd083]

\[    0.389976] Detected PIPT I-cache on CPU2

\[    0.391514] cacheinfo: Unable to detect cache hierarchy for CPU 2

\[    0.393306] CPU2: Booted secondary processor 0x0000000002 \[0x410fd083]

\[    0.407774] Detected PIPT I-cache on CPU3

\[    0.409082] cacheinfo: Unable to detect cache hierarchy for CPU 3

\[    0.410612] CPU3: Booted secondary processor 0x0000000003 \[0x410fd083]

\[    0.420264] Detected PIPT I-cache on CPU4

\[    0.421619] cacheinfo: Unable to detect cache hierarchy for CPU 4

\[    0.423138] CPU4: Booted secondary processor 0x0000000004 \[0x410fd083]

\[    0.432418] Detected PIPT I-cache on CPU5

\[    0.433736] cacheinfo: Unable to detect cache hierarchy for CPU 5

\[    0.435197] CPU5: Booted secondary processor 0x0000000005 \[0x410fd083]

\[    0.444941] Detected PIPT I-cache on CPU6

\[    0.446466] cacheinfo: Unable to detect cache hierarchy for CPU 6

\[    0.448048] CPU6: Booted secondary processor 0x0000000006 \[0x410fd083]

\[    0.458291] Detected PIPT I-cache on CPU7

\[    0.459889] cacheinfo: Unable to detect cache hierarchy for CPU 7

\[    0.461603] CPU7: Booted secondary processor 0x0000000007 \[0x410fd083]

\[    0.464408] smp: Brought up 1 node, 8 CPUs

\[    0.465358] SMP: Total of 8 processors activated.

\[    0.465937] CPU features: detected: 32-bit EL0 Support

\[    0.466108] CPU features: detected: 32-bit EL1 Support

\[    0.466659] CPU features: detected: CRC32 instructions

\[    0.521227] CPU: All CPU(s) started at EL1

\[    0.522007] alternatives: applying system-wide alternatives

\[    0.665011] devtmpfs: initialized

\[    0.757308] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns

\[    0.759494] futex hash table entries: 2048 (order: 5, 131072 bytes, linear)

\[    0.780097] pinctrl core: initialized pinctrl subsystem

\[    0.845433] DMI not present or invalid.

\[    0.894497] NET: Registered PF_NETLINK/PF_ROUTE protocol family

\[    0.964930] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations

\[    0.969401] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations

\[    0.971682] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations

\[    0.973403] audit: initializing netlink subsys (disabled)

\[    0.990866] audit: type=2000 audit(0.772:1): state=initialized audit_enabled=0 res=1

\[    1.017865] thermal_sys: Registered thermal governor 'step_wise'

\[    1.018151] thermal_sys: Registered thermal governor 'power_allocator'

\[    1.020840] cpuidle: using governor menu

\[    1.027958] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.

\[    1.036769] ASID allocator initialised with 32768 entries

\[    1.072882] Serial: AMBA PL011 UART driver

\[    1.296562] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 13, base_baud = 0) is a PL011 rev1

\[    1.503418] printk: console \[ttyAMA0] enabled

\[    1.572052] KASLR enabled

\[    1.734648] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages

\[    1.736199] HugeTLB: 16380 KiB vmemmap can be freed for a 1.00 GiB page

\[    1.737757] HugeTLB: registered 32.0 MiB page size, pre-allocated 0 pages

\[    1.739926] HugeTLB: 508 KiB vmemmap can be freed for a 32.0 MiB page

\[    1.740902] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages

\[    1.742456] HugeTLB: 28 KiB vmemmap can be freed for a 2.00 MiB page

\[    1.744922] HugeTLB: registered 64.0 KiB page size, pre-allocated 0 pages

\[    1.746089] HugeTLB: 0 KiB vmemmap can be freed for a 64.0 KiB page

\[    1.808152] ACPI: Interpreter disabled.

\[    1.848039] iommu: Default domain type: Translated

\[    1.849913] iommu: DMA domain TLB invalidation policy: strict mode

\[    1.859372] SCSI subsystem initialized

\[    1.875374] usbcore: registered new interface driver usbfs

\[    1.878273] usbcore: registered new interface driver hub

\[    1.882311] usbcore: registered new device driver usb

\[    1.897213] pps_core: LinuxPPS API ver. 1 registered

\[    1.898369] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti \<giometti@linux.it>

\[    1.901883] PTP clock support registered

\[    1.905231] EDAC MC: Ver: 3.0.0

\[    1.940091] FPGA manager framework

\[    1.945096] Advanced Linux Sound Architecture Driver Initialized.

\[    2.002642] vgaarb: loaded

\[    2.091444] clocksource: Switched to clocksource arch_sys_counter

\[    2.117444] VFS: Disk quotas dquot_6.6.0

\[    2.119794] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)

\[    2.130285] pnp: PnP ACPI: disabled

\[    2.310575] NET: Registered PF_INET protocol family

\[    2.318174] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)

\[    2.358581] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)

\[    2.360924] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)

\[    2.362520] TCP established hash table entries: 1024 (order: 1, 8192 bytes, linear)

\[    2.365462] TCP bind hash table entries: 1024 (order: 3, 32768 bytes, linear)

\[    2.367582] TCP: Hash tables configured (established 1024 bind 1024)

\[    2.443555] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)

\[    2.446309] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)

\[    2.455380] NET: Registered PF_UNIX/PF_LOCAL protocol family

\[    2.479354] RPC: Registered named UNIX socket transport module.

\[    2.480917] RPC: Registered udp transport module.

\[    2.481804] RPC: Registered tcp transport module.

\[    2.482539] RPC: Registered tcp NFSv4.1 backchannel transport module.

\[    2.483899] PCI: CLS 0 bytes, default 64

\[    2.522041] Unpacking initramfs...

\[    2.539734] hw perfevents: enabled with armv8_pmuv3 PMU driver, 7 counters available

\[    2.554091] kvm \[1]: HYP mode not available

\[    2.594326] Initialise system trusted keyrings

\[    2.605053] workingset: timestamp_bits=42 max_order=15 bucket_order=0

\[    2.751485] squashfs: version 4.0 (2009/01/31) Phillip Lougher

\[    2.778069] NFS: Registering the id_resolver key type

\[    2.781289] Key type id_resolver registered

\[    2.782194] Key type id_legacy registered

\[    2.786292] nfs4filelayout_init: NFSv4 File Layout Driver Registering...

\[    2.788757] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...

\[    2.794800] 9p: Installing v9fs 9p2000 file system support

\[    2.962624] Key type asymmetric registered

\[    2.963954] Asymmetric key parser 'x509' registered

\[    2.967160] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 245)

\[    2.969632] io scheduler mq-deadline registered

\[    2.970791] io scheduler kyber registered

\[    3.023779] Freeing initrd memory: 1212K

\[    3.193338] pl061_gpio 9030000.pl061: PL061 GPIO chip registered

\[    3.232680] pci-host-generic 4010000000.pcie: host bridge /pcie@10000000 ranges:

\[    3.237400] pci-host-generic 4010000000.pcie:       IO 0x003eff0000..0x003effffff -> 0x0000000000

\[    3.241782] pci-host-generic 4010000000.pcie:      MEM 0x0010000000..0x003efeffff -> 0x0010000000

\[    3.243432] pci-host-generic 4010000000.pcie:      MEM 0x8000000000..0xffffffffff -> 0x8000000000

\[    3.248984] pci-host-generic 4010000000.pcie: Memory resource size exceeds max for 32 bits

\[    3.253888] pci-host-generic 4010000000.pcie: ECAM at \[mem 0x4010000000-0x401fffffff] for \[bus 00-ff]

\[    3.265394] pci-host-generic 4010000000.pcie: PCI host bridge to bus 0000:00

\[    3.267974] pci_bus 0000:00: root bus resource \[bus 00-ff]

\[    3.269976] pci_bus 0000:00: root bus resource \[io  0x0000-0xffff]

\[    3.271170] pci_bus 0000:00: root bus resource \[mem 0x10000000-0x3efeffff]

\[    3.272908] pci_bus 0000:00: root bus resource \[mem 0x8000000000-0xffffffffff]

\[    3.283711] pci 0000:00:00.0: \[1b36:0008] type 00 class 0x060000

\[    3.310041] pci 0000:00:01.0: \[8086:100e] type 00 class 0x020000

\[    3.312601] pci 0000:00:01.0: reg 0x10: \[mem 0x00000000-0x0001ffff]

\[    3.314140] pci 0000:00:01.0: reg 0x14: \[io  0x0000-0x003f]

\[    3.315230] pci 0000:00:01.0: reg 0x30: \[mem 0x00000000-0x0003ffff pref]

\[    3.331329] pci 0000:00:01.0: BAR 6: assigned \[mem 0x10000000-0x1003ffff pref]

\[    3.334502] pci 0000:00:01.0: BAR 0: assigned \[mem 0x10040000-0x1005ffff]

\[    3.335946] pci 0000:00:01.0: BAR 1: assigned \[io  0x1000-0x103f]

\[    3.376146] EINJ: ACPI disabled.

\[    6.811704] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled

\[    6.862840] SuperH (H)SCI(F) driver initialized

\[    6.871494] msm_serial: driver initialized

\[    6.901232] cacheinfo: Unable to detect cache hierarchy for CPU 0

\[    7.086548] loop: module loaded

\[    7.107737] megasas: 07.719.03.00-rc1

\[    7.146483] physmap-flash 0.flash: physmap platform flash device: \[mem 0x00000000-0x03ffffff]

\[    7.156826] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000

\[    7.161047] Intel/Sharp Extended Query Table at 0x0031

\[    7.166601] Using buffer write method

\[    7.170663] physmap-flash 0.flash: physmap platform flash device: \[mem 0x04000000-0x07ffffff]

\[    7.175718] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000

\[    7.178200] Intel/Sharp Extended Query Table at 0x0031

\[    7.183545] Using buffer write method

\[    7.186232] Concatenating MTD devices:

\[    7.187173] (0): "0.flash"

\[    7.187838] (1): "0.flash"

\[    7.188627] into device "0.flash"

\[    7.652724] tun: Universal TUN/TAP device driver, 1.6

\[    7.720986] thunder_xcv, ver 1.0

\[    7.723589] thunder_bgx, ver 1.0

\[    7.725816] nicpf, ver 1.0

\[    7.757131] hns3: Hisilicon Ethernet Network Driver for Hip08 Family - version

\[    7.758568] hns3: Copyright (c) 2017 Huawei Corporation.

\[    7.762237] hclge is initializing

\[    7.763635] igb: Intel(R) Gigabit Ethernet Network Driver

\[    7.765260] igb: Copyright (c) 2007-2014 Intel Corporation.

\[    7.767005] igbvf: Intel(R) Gigabit Virtual Function Network Driver

\[    7.768812] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.

\[    7.776907] sky2: driver version 1.30

\[    7.797884] VFIO - User Level meta-driver version: 0.3

\[    7.859380] usbcore: registered new interface driver usb-storage

\[    7.935973] rtc-pl031 9010000.pl031: registered as rtc0

\[    7.941032] rtc-pl031 9010000.pl031: setting system clock to 2023-11-18T09:16:03 UTC (1700298963)

\[    7.955616] i2c_dev: i2c /dev entries driver

\[    8.059070] sdhci: Secure Digital Host Controller Interface driver

\[    8.059855] sdhci: Copyright(c) Pierre Ossman

\[    8.071777] Synopsys Designware Multimedia Card Interface Driver

\[    8.087867] sdhci-pltfm: SDHCI platform and OF driver helper

\[    8.124867] ledtrig-cpu: registered to indicate activity on CPUs

\[    8.161537] usbcore: registered new interface driver usbhid

\[    8.162572] usbhid: USB HID core driver

\[    8.277153] NET: Registered PF_PACKET protocol family

\[    8.287369] 9pnet: Installing 9P2000 support

\[    8.290182] Key type dns_resolver registered

\[    8.299244] registered taskstats version 1

\[    8.301883] Loading compiled-in X.509 certificates

\[    8.579201] input: gpio-keys as /devices/platform/gpio-keys/input/input0

\[    8.633746] ALSA device list:

\[    8.635656]   No soundcards found.

\[    8.657956] uart-pl011 9000000.pl011: no DMA platform data

\[    9.316736] Freeing unused kernel memory: 1920K

\[    9.359573] Run /init as init process

\[    9.859655] e1000_for_linux: loading out-of-tree module taints kernel.

\[    9.912870] rust_e1000dev: Rust e1000 device driver (init)

\[    9.922525] rust_e1000dev: PCI Driver probing Some(1)

\[    9.929381] rust_e1000dev 0000:00:01.0: enabling device (0000 -> 0003)

\[    9.936856] rust_e1000dev: PCI MappedResource addr: 0xffff800008480000, len: 131072, irq: 19

\[    9.951156] rust_e1000dev: get stats64

\[   10.065963] rust_e1000dev: get stats64

eth1      Link encap:Ethernet  HWaddr 52:54:00:12:34:55

          BROADCAST MULTICAST  MTU:1500  Metric:1

          RX packets:0 errors:0 dropped:0 overruns:0 frame:0

          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0

          collisions:0 txqueuelen:1000

          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)



\[   10.236884] rust_e1000dev: Ethernet E1000 open

\[   10.237899] rust_e1000dev: New E1000 device @ 0xffff800008480000

\[   10.241513] rust_e1000dev: Allocated vaddr: 0xffff18b08332f000, paddr: 0x4332f000

\[   10.244159] rust_e1000dev: Allocated vaddr: 0xffff18b083424000, paddr: 0x43424000

\[   10.258288] rust_e1000dev: Allocated vaddr: 0xffff18b085c80000, paddr: 0x45c80000

\[   10.267599] rust_e1000dev: Allocated vaddr: 0xffff18b085d00000, paddr: 0x45d00000

\[   10.270784] rust_e1000dev: e1000 CTL: 0x140240, Status: 0x80080783

\[   10.271943] rust_e1000dev: e1000_init has been completed

\[   10.273726] rust_e1000dev: e1000 device is initialized

\[   10.280992] rust_e1000dev: handle_irq

\[   10.282869] rust_e1000dev: irq::Handler E1000_ICR = 0x4

\[   10.286218] rust_e1000dev: get stats64

\[   10.289874] rust_e1000dev: NapiPoller poll

\[   10.291886] rust_e1000dev: e1000_recv

\[   10.291886]

\[   10.292258] rust_e1000dev: irq::handler recv is None

\[   10.323775] rust_e1000dev: get stats64

/bin/sh: can't access tty; job control turned off

/ # ping 10.0.2.2

PING 10.0.2.2 (10.0.2.2): 56 data bytes

\[   18.279362] \[r4l test]:binding start xmit

\[   18.281663] rust_e1000dev: Read E1000_TDT = 0x0

\[   18.282078] rust_e1000dev: >>>>>>>>> TX PKT 60

\[   18.282844] rust_e1000dev:

\[   18.282844]

\[   18.284086] rust_e1000dev: handle_irq

\[   18.291407] rust_e1000dev: irq::Handler E1000_ICR = 0x83

\[   18.293517] rust_e1000dev: NapiPoller poll

\[   18.295755] rust_e1000dev: Read E1000_RDT + 1 = 0x0

\[   18.296219] rust_e1000dev: RX PKT 64 \<\<\<\<\<\<\<\<\<

\[   18.299517] rust_e1000dev: e1000_recv

\[   18.299517]

\[   18.301622] rust_e1000dev: irq::Handler recv = 0x1

\[   18.316187] rust_e1000dev: irq::Handler recv = packets_len:0x1 datalen:0x40

\[   18.330359] \[r4l test]:binding start xmit

\[   18.332177] rust_e1000dev: Read E1000_TDT = 0x1

\[   18.332326] rust_e1000dev: >>>>>>>>> TX PKT 98

\[   18.334101] rust_e1000dev:

\[   18.334101]

\[   18.338608] rust_e1000dev: handle_irq

\[   18.340693] rust_e1000dev: irq::Handler E1000_ICR = 0x83

\[   18.346517] rust_e1000dev: NapiPoller poll

\[   18.347330] rust_e1000dev: Read E1000_RDT + 1 = 0x1

\[   18.347421] rust_e1000dev: RX PKT 98 \<\<\<\<\<\<\<\<\<

\[   18.348343] rust_e1000dev: e1000_recv

\[   18.348343]

\[   18.350169] rust_e1000dev: irq::Handler recv = 0x1

\[   18.353030] rust_e1000dev: irq::Handler recv = packets_len:0x1 datalen:0x62

64 bytes from 10.0.2.2: seq=0 ttl=255 time=122.904 ms

\[   19.313489] \[r4l test]:binding start xmit

\[   19.315003] rust_e1000dev: Read E1000_TDT = 0x2

\[   19.315273] rust_e1000dev: >>>>>>>>> TX PKT 98

\[   19.316666] rust_e1000dev:

\[   19.316666]

\[   19.318228] rust_e1000dev: handle_irq

\[   19.323728] rust_e1000dev: irq::Handler E1000_ICR = 0x83

\[   19.325865] rust_e1000dev: NapiPoller poll

\[   19.327389] rust_e1000dev: Read E1000_RDT + 1 = 0x2

\[   19.327518] rust_e1000dev: RX PKT 98 \<\<\<\<\<\<\<\<\<

\[   19.329009] rust_e1000dev: e1000_recv

\[   19.329009]

\[   19.330706] rust_e1000dev: irq::Handler recv = 0x1

\[   19.333860] rust_e1000dev: irq::Handler recv = packets_len:0x1 datalen:0x62

64 bytes from 10.0.2.2: seq=1 ttl=255 time=27.249 ms

\[   20.339028] \[r4l test]:binding start xmit

\[   20.340420] rust_e1000dev: Read E1000_TDT = 0x3

\[   20.340669] rust_e1000dev: >>>>>>>>> TX PKT 98

\[   20.341640] rust_e1000dev:

\[   20.341640]

\[   20.343304] rust_e1000dev: handle_irq

\[   20.346551] rust_e1000dev: irq::Handler E1000_ICR = 0x83

\[   20.348644] rust_e1000dev: NapiPoller poll

\[   20.350200] rust_e1000dev: Read E1000_RDT + 1 = 0x3

\[   20.350338] rust_e1000dev: RX PKT 98 \<\<\<\<\<\<\<\<\<

\[   20.352735] rust_e1000dev: e1000_recv

\[   20.352735]

\[   20.354760] rust_e1000dev: irq::Handler recv = 0x1

\[   20.358279] rust_e1000dev: irq::Handler recv = packets_len:0x1 datalen:0x62

64 bytes from 10.0.2.2: seq=2 ttl=255 time=23.375 ms
