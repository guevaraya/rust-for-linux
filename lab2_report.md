## 练习2 自定义编写Rust内核驱动模块

* 1. linux目录下samples/rust，新建rust_helloworld.rs  并添加对应的Kconfig
     **rust_helloworld.rs** 


```
// SPDX-License-Identifier: GPL-2.0
//! Rust minimal sample.
      
use kernel::prelude::*;
      
module! {
  type: RustHelloWorld,
  name: "rust_helloworld",
  author: "whocare",
  description: "hello world module in rust",
  license: "GPL",
}
      
struct RustHelloWorld {}
      
impl kernel::Module for RustHelloWorld {
  fn init(_name: &'static CStr, _module: &'static ThisModule) -> Result<Self> {
      pr_info!("Hello World from Rust module");
      Ok(RustHelloWorld {})
  }
}

```

* 1. samples/rust/Kconfig 配置修改如下：


```
obj-$(CONFIG_SAMPLE_RUST_FS)            += rust_fs.o
obj-$(CONFIG_SAMPLE_RUST_SELFTESTS)        += rust_selftests.o
      // ++++++++++++++++ add here
obj-$(CONFIG_SAMPLE_RUST_HELLOWORLD)        += rust_helloworld.o
      // ++++++++++++++++
subdir-$(CONFIG_SAMPLE_RUST_HOSTPROGS)        += hostprogs
  
config SAMPLE_RUST_MINIMAL
  tristate "Minimal"
  help
    This option builds the Rust minimal module sample.
      
    To compile this as a module, choose M here:
    the module will be called rust_minimal.
      
    If unsure, say N.
      // +++++++++++++++ add here
config SAMPLE_RUST_HELLOWORLD
  tristate "Print Helloworld in Rust"
  help
    This option builds the Rust HelloWorld module sample.
      
    To compile this as a module, choose M here:
    the module will be called rust_helloworld.
      
    If unsure, say N.
      // +++++++++++++++
config SAMPLE_RUST_PRINT
  tristate "Printing macros"
  help
    This option builds the Rust printing macros sample.
      
    To compile this as a module, choose M here:
    the module will be called rust_print.
      
    If unsure, say N.

```

3，配置使能 run make LLVM=1 menuconfig

```
Kernel hacking
  ---> Sample Kernel code
      ---> Rust samples
              ---> <*>Print Helloworld in Rust (NEW)

```

4，编译内核 run make  LLVM=1

5, 启动QEMU,

为了方便直接在<https://people.debian.org/~gio/dqib/> 这里下载根文件系统，解压后参照readme.txt，更换自己的内核后，运行QEMU目录：

```
qemu-system-aarch64 -machine 'virt' -cpu 'cortex-a57' -m 1G -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 -kernel ../linux/build/arch/arm64/boot/Image -initrd initrd -nographic -append "root=LABEL=rootfs console=ttyAMA0"

```

### 使用 busybox 制作内存文件系统 initramfs：

1. 下载解压 busybox 并配置环境变量

   ````
   ```shell
   wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
   tar -xf busybox-1.35.0.tar.bz2
   cd busybox-1.35.0
   # 配置环境变量
   export ARCH=arm64
   export CROSS_COMPILE=aarch64-linux-gnu-
   ```

   ````

   如果教程工具没有安装，请用下面命令


```
sudo apt-get install gcc-aarch64-linux-gnu

```

1. 配置编译内核的参数

   ```shell
   # busybox-1.35.0目录下
   make menuconfig
   # 修改配置，选中如下项目，静态编译
   # Settings -> Build Options -> [*] Build static binary (no share libs)

   ```


1. 编译

   ```shell
   make -j `nproc`

   ```


```
编译头文件出错，错误如下：
make -j `nproc`
  SPLIT   include/autoconf.h -> include/config/*
  GEN     include/bbconfigopts.h
  GEN     include/common_bufsiz.h
  GEN     include/embedded_scripts.h
  HOSTCC  applets/usage
  HOSTCC  applets/applet_tables
  GEN     include/usage_compressed.h
  GEN     include/applet_tables.h include/NUM_APPLETS.h
  GEN     include/applet_tables.h include/NUM_APPLETS.h
  HOSTCC  applets/usage_pod
  CC      applets/applets.o
In file included from /usr/lib/gcc-cross/aarch64-linux-gnu/10/include/limits.h:195,
                 from /usr/lib/gcc-cross/aarch64-linux-gnu/10/include/syslimits.h:7,
                 from /usr/lib/gcc-cross/aarch64-linux-gnu/10/include/limits.h:34,
                 from include/platform.h:157,
                 from include/libbb.h:13,
                 from include/busybox.h:8,
                 from applets/applets.c:9:
/usr/include/limits.h:26:10: fatal error: bits/libc-header-start.h: 没有那个文件或目录
   26 | #include <bits/libc-header-start.h>
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
make[1]: *** [scripts/Makefile.build:198：applets/applets.o] 错误 1
make: *** [Makefile:372：applets_dir] 错误 2

```

需要安装apt install gcc-multilib

```
 make
  CC      applets/applets.o
In file included from /usr/include/bits/errno.h:26,
                 from /usr/include/errno.h:28,
                 from include/libbb.h:17,
                 from include/busybox.h:8,
                 from applets/applets.c:9:
/usr/include/linux/errno.h:1:10: fatal error: asm/errno.h: 没有那个文件或目录
    1 | #include <asm/errno.h>
      |          ^~~~~~~~~~~~~
compilation terminated.
make[1]: *** [scripts/Makefile.build:198：applets/applets.o] 错误 1
make: *** [Makefile:372：applets_dir] 错误 2

```

解决方法：sudo ln -s /usr/i686-linux-gnu/include/asm /usr/include/asm

1. 安装

   安装前 busybox-1.35.0 目录下的文件如下图所示

输入如下命令

````
```shell
make install
```


安装后目录下生成了\_install 目录

````

1. 在\_install 目录下创建后续所需的文件和目录

   ```shell
   cd _install
   mkdir proc sys dev tmp
   touch init
   chmod +x init

   ```

2. 用任意的文本编辑器编辑 init 文件内容如下

   ```shell
   #!/bin/sh

   # 挂载一些必要的文件系统
   mount -t proc none /proc
   mount -t sysfs none /sys
   mount -t tmpfs none /tmp
   mount -t devtmpfs none /dev

   ```


````
# 停留在控制台
exec /bin/sh
```

````

1. 用 busybox 制作 initramfs 文件

   ```shell
   # _install目录
   find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz

   ```

   执行成功后可在 busybox-1.35.0 目录下找到 initramfs.cpio.gz 文件

2. 进入 qemu 目录下，执行如下命令

   ````
   ```shell
   qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel (your Image path) -initrd (your initramfs.cpio.gz path) -nographic -append "init=/init console=ttyAMA0"
   ```

   其中`(your Image path)`为上一个任务中最后的`Image`镜像文件所在的目录，`(your initramfs.cpio.gz)`为步骤 7 中执行成功后得到的 initramfs.cpio.gz 的目录，例如

   ```shell
   qemu-system-aarch64 -M virt -cpu cortex-a72 -smp 8 -m 128M -kernel /home/jun/maodou/linux/arch/arm64/boot/Image -initrd /home/jun/maodou/busybox-1.35.0/initramfs.cpio.gz -nographic -append "init=/init console=ttyAMA0"
   ```

   ````

   \*问题：failed to find romfile "efi-virtio.rom"
   解决：apt-get install ipxe-qemu
       
![image](https://github.com/guevaraya/rust-for-linux/assets/446973/c2fc4220-df5f-4353-a07d-79381223d87d)

【1】[QEMU安装启动实例](http://iric.tpddns.cn:9955/#/Rust/2023%E7%A7%8B%E5%86%AC%E8%AE%AD%E7%BB%83%E8%90%A5/%E7%AC%AC%E4%B8%89%E9%98%B6%E6%AE%B5/rust-for-linux/Qemu%E6%A8%A1%E6%8B%9F%E5%90%AF%E5%8A%A8/README)

[\[2\]  编译 linux for rust 并制作 initramfs 最后编写 rust_helloworld 内核驱动 并在 qemu 环境加载](https://blog.csbxd.fun/archives/1699538198511)
[通过 QEMU 打开学习 Linux kernel 的新世界](https://www.jianshu.com/p/9b68e9ea5849)

````
```

````

