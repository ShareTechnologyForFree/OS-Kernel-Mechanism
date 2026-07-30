# 编译linux内核并开启gdb调试

# 1、编译linux内核

## 1.1、生成默认配置文件

```bash
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
```

## 1.2、打开图形界面进行参数配置

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```

```c
	General setup 
		[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support  

	Device Drivers  ---> 
		[*] Block devices  --->  
			<*>   RAM block device support 
				(65536) Default RAM disk size (kbytes  // 使用deconfig使用这一项就行

	Kernel hacking  ---> 
		Compile-time checks and compiler options  ---> 
			Debug information (Rely on the toolchain's implicit default DWARF version)  --->  
				(X) Rely on the toolchain's implicit default DWARF version 

	Device Drivers  ---> 
		[*] Network device support  ---> 
			<*>     Universal TUN/TAP device driver support 

	[*] Networking support  ---> 
		Networking options  ---> 
			<M> 802.1d Ethernet Bridging
```

## 1.3、编译内核镜像和模块

```bash
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image -j$(nproc) 
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules -j$(nproc) 
```

## 1.4、编译ko出错解决方案

```bash
zdy@DESKTOP-3QNUG9S ~/test
$ make vabits
make -C /home/zdy/linux_old1 M=/home/zdy/test ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- obj-m=vabits_test.o
make[1]: Entering directory '/home/zdy/linux_old1'
make[2]: Entering directory '/home/zdy/test'
***
***  ERROR: Kernel configuration is invalid. The following files are missing:
***    - /home/zdy/linux_old1/include/generated/autoconf.h
***    - /home/zdy/linux_old1/include/generated/rustc_cfg
***  Run "make oldconfig && make prepare" on kernel source to fix it.
***
make[3]: *** [/home/zdy/linux_old1/Makefile:853: /home/zdy/linux_old1/include/config/auto.conf] Error 1
make[2]: *** [/home/zdy/linux_old1/Makefile:251: __sub-make] Error 2
make[2]: Leaving directory '/home/zdy/test'
make[1]: *** [Makefile:251: __sub-make] Error 2
make[1]: Leaving directory '/home/zdy/linux_old1'
make: *** [Makefile:27: vabits] Error 2

zdy@DESKTOP-3QNUG9S ~/test
$
```

上述编译ko文件出错，进入kernel目录：

```bash
zdy@DESKTOP-3QNUG9S ~/test
$ cd ../linux_old1/

zdy@DESKTOP-3QNUG9S ~/linux_old1
[v6.14]$ sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- prepare
[sudo] password for zdy:
  SYNC    include/config/auto.conf.cmd
  CC      arch/arm64/kernel/asm-offsets.s
  CALL    scripts/checksyscalls.sh
  AS      arch/arm64/kernel/vdso/note.o
  AS      arch/arm64/kernel/vdso/sigreturn.o
  AS      arch/arm64/kernel/vdso/vgetrandom-chacha.o
  LD      arch/arm64/kernel/vdso/vdso.so.dbg
  VDSOSYM include/generated/vdso-offsets.h
  OBJCOPY arch/arm64/kernel/vdso/vdso.so

zdy@DESKTOP-3QNUG9S ~/linux_old1
[v6.14]$
```

再次编译即可：

```bash
zdy@DESKTOP-3QNUG9S ~/linux_old1
[v6.14]$ cd ../test/

zdy@DESKTOP-3QNUG9S ~/test
$ make all
make -C /home/zdy/linux_old1 M=/home/zdy/test ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
make[1]: Entering directory '/home/zdy/linux_old1'
make[2]: Entering directory '/home/zdy/test'
  CC [M]  buddy_test.o
  CC [M]  slab_test.o
  CC [M]  kmalloc_test.o
  CC [M]  vabits_test.o
  MODPOST Module.symvers
  CC [M]  buddy_test.mod.o
  CC [M]  .module-common.o
  LD [M]  buddy_test.ko
  CC [M]  slab_test.mod.o
  LD [M]  slab_test.ko
  CC [M]  kmalloc_test.mod.o
  LD [M]  kmalloc_test.ko
  CC [M]  vabits_test.mod.o
  LD [M]  vabits_test.ko
make[2]: Leaving directory '/home/zdy/test'
make[1]: Leaving directory '/home/zdy/linux_old1'

zdy@DESKTOP-3QNUG9S ~/test
$ ls
Makefile        buddy_test.mod    kmalloc_test.c      kmalloc_test.mod.o  slab_test.ko     slab_test.o      vabits_test.mod.c
Module.symvers  buddy_test.mod.c  kmalloc_test.ko     kmalloc_test.o      slab_test.mod    vabits_test.c    vabits_test.mod.o
buddy_test.c    buddy_test.mod.o  kmalloc_test.mod    modules.order       slab_test.mod.c  vabits_test.ko   vabits_test.o
buddy_test.ko   buddy_test.o      kmalloc_test.mod.c  slab_test.c         slab_test.mod.o  vabits_test.mod

zdy@DESKTOP-3QNUG9S ~/test
$
```

# 2、根文件系统制作

## 2.1、下载

​	wget https://www.busybox.net/downloads/busybox-1.36.1.tar.bz2

## 2.2、解压源码

​	tar -xjf busybox-1.36.1.tar.bz2

## 2.3、移除 networking/tc.c ，解决编译报错

​	mv networking/tc.c ../

## 2.4、配置busybox源码

make menuconfig

	Settings  --->  
		[*] Build static binary (no shared libs) 
	
	Settings  --->  
		(aarch64-linux-gnu-) Cross compiler prefix

## 2.5、编译和安装

sudo make
sudo make install

## 2.6、创建必要文件

编译完成后的busybox就安装在源码根目录下的_install目录中，进入_install目录补充一些必要的文件和目录
mkdir -p etc dev mnt proc sys tmp etc/init.d

sudo vim etc/fstab

	proc    /proc   proc    defaults        0       0
	tmpfs   /proc   tmpfs   defaults        0       0
	sysfs   /sys    sysfs   defaults        0       0

sudo vim etc/init.d/rcS

	/bin/mount -a
	mount -o remount,rw /
	mkdir -p /dev/pts
	mount -t devpts devpts /dev/pts
	echo /sbin/mdev > /proc/sys/kernel/hotplug
	mdev -s
sudo chmod 755 etc/init.d/rcS

sudo vim etc/inittab

	::sysinit:/etc/init.d/rcS
	::respawn:-/bin/sh
	::askfirst:-/bin/sh
	::ctrlaltdel:/bin/umount -a -r
sudo chmod 755 etc/inittab

sudo vim etc/profile

mknod	console	c	5	1
mknod	null	c	1	3
mknod	tty1	c	4	1

## 2.7、制作根文件系统镜像文件

- 先制作一个空的镜像文件；
- 然后把此镜像文件格式化为ext3格式；
- 然后把此镜像文件挂载，并把根文件系统复制到挂载目录；
- 卸载该镜像文件。
- 打成gzip包。

cd busybox-1.36.1/

sudo vim build_img.sh

	#!/bin/bash
	rm -rf rootfs.ext3
	rm -rf fs
	dd if=/dev/zero of=./rootfs.ext3 bs=1M count=32
	mkfs.ext3 rootfs.ext3
	mkdir fs
	mount -o loop rootfs.ext3 ./fs
	cp -rf ./_instal1/* ./fs
	umount ./fs
	gzip --best -c rootfs.ext3 > rootfs.img.gz
sudo chmod 775 build_img.sh

sudo ./build_img.sh

	// 生成 rootfs.img.gz

## 2.8、创建 tap0

sudo ip tuntap add tap0 mode tap
sudo ip link set tap0 promisc on
sudo ip link set tap0 up
sudo brctl addif br0 tap0

- 检查现有网桥	sudo brctl show				确认系统中当前所有的网桥，验证 br0是否真的不存在。
- 创建网桥 br0	sudo brctl addbr br0		创建一个名为 br0的新网桥设备 。
- 添加接口到网桥	sudo brctl addif br0 tap0	将网络接口 tap0添加到网桥 br0中 。
- 启用网桥		sudo ip link set dev br0 up	启动 br0网桥设备，使其开始工作。
- 查看网桥		sudo brctl show				查看当前系统上已经存在的网桥列表。

# 3、启动aarch64的linux内核

```bash
sudo brctl addbr br0
sudo brctl addif br0 tap0
sudo ip link set dev br0 up
sudo brctl show
cd ~/aarch64_linux
sudo qemu-system-aarch64 -machine virt,virtualization=true,gic-version=3 -nographic -m size=1024M -cpu cortex-a57 -smp 4 -kernel ~/linux_old1/arch/arm64/boot/Image -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8" -net nic -net tap,ifname=tap0,script=no
```

配置网络，这块暂时不通

ifconfig 

​		eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
​				inet 192.168.188.143  net

	// 在启动的qemu当中设置IP地址		
	~ # ifconfig eth0 192.168.188.111

# 4、安装 gdb

sudo apt install gdb-multiarch

# 5、VSCode的gdb配置

我使用的是CodeBuddy，打开WSL中的 ~/linux_old1 文件夹
新建 .vscode/launch.json 文件，内容如下：

```json
	{
		"version":"0.2.0",
		"configurations": [
			{
				"name":"kernel-debug",
				"type":"cppdbg",
				"request":"launch",
				"miDebuggerServerAddress":"127.0.0.1:1234",
				"program":"${workspaceFolder}/vmlinux",
				"args": [],
				"stopAtEntry":false,
				"cwd":"${workspaceFolder}",
				"environment":[],
				"externalConsole":false,
				"logging":{
					"engineLogging":false
				},
				"MIMode":"gdb",
				"miDebuggerPath":"/usr/bin/gdb-multiarch",
				"targetArchitecture":"arm64",
			}
		]
	}
```

# 6、执行gdb

1）WSL中启动qemu，添加 nokaslr 和 -S -s 

```bash
cd ~/aarch64_linux && qemu-system-aarch64 -machine virt,virtualization=true,gic-version=3 -nographic -m size=1024M -cpu cortex-a57 -smp 4 -kernel ~/linux_old1/arch/arm64/boot/Image -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8 nokaslr" -net nic -net tap,ifname=tap0,script=no -S -s
```


![image-20260131235813166](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260131235813166.png)

2）CodeBuddy中 

* 安装 @category:debuggers cppdbg”, 插件 并运行 --> 启动调试

![image-20260131235958015](C:\Users\Daniel\AppData\Roaming\Typora\typora-user-images\image-20260131235958015.png)



# 7、模拟UMA、NUMA

## 7.1、UMA

```bash
sudo qemu-system-aarch64 -machine virt,virtualization=true,gic-version=3 -nographic -m size=1024M -cpu cortex-a57 -smp 4 -kernel ~/linux_old1/arch/arm64/boot/Image -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8"

// 开启UMA，一个Node，不调试
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m size=1024M \
  -cpu cortex-a57 \
  -smp 4 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz \
  -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8"
  
// 这个是修复一些rootfs、root=/dev/ram 之后的版本  
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m size=1024M \
  -cpu cortex-a57 \
  -smp 4 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.cpio.gz \
  -append "console=ttyAMA0 rdinit=/linuxrc loglevel=8"  
  
 // 开启UMA，一个Node，调试
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m size=1024M \
  -cpu cortex-a57 \
  -smp 4 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz \
  -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8 nokaslr" -S -s

```

启动之后：

```bash
Please press Enter to activate this console.
~ #
~ #
~ # cat /proc/cpuinfo
processor       : 0
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 1
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 2
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 3
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

~ # cat /proc/zoneinfo | grep Node
Node 0, zone      DMA
Node 0, zone    DMA32
Node 0, zone   Normal
Node 0, zone  Movable
~ #
```

## 7.2、NUMA

```bash
sudo qemu-system-aarch64 -machine virt,virtualization=true,gic-version=3 -nographic -m 2G -cpu cortex-a57 -smp cores=8,threads=1 -object memory-backend-ram,id=mem0,size=1G -object memory-backend-ram,id=mem1,size=1G -numa node,memdev=mem0,cpus=0-3,nodeid=0 -numa node,memdev=mem1,cpus=4-7,nodeid=1 -kernel ~/linux_old1/arch/arm64/boot/Image -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8"

// 开启NUMA，两个Node，不调试
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m 2G \
  -cpu cortex-a57 \
  -smp cores=8,threads=1 \
  -object memory-backend-ram,id=mem0,size=1G \
  -object memory-backend-ram,id=mem1,size=1G \
  -numa node,memdev=mem0,cpus=0-3,nodeid=0 \
  -numa node,memdev=mem1,cpus=4-7,nodeid=1 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz \
  -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8"

// 开启NUMA，两个Node，调试
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m 2G \
  -cpu cortex-a57 \
  -smp cores=8,threads=1 \
  -object memory-backend-ram,id=mem0,size=1G \
  -object memory-backend-ram,id=mem1,size=1G \
  -numa node,memdev=mem0,cpus=0-3,nodeid=0 \
  -numa node,memdev=mem1,cpus=4-7,nodeid=1 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz \
  -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8 nokaslr" -S -s
```

启动之后：

```bash
Please press Enter to activate this console.
~ #
~ # ls
bin         etc         mnt         sbin        tty1
console     linuxrc     null        sys         usr
dev         lost+found  proc        tmp
~ #
~ #
~ # cat /proc/cpuinfo
processor       : 0
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 1
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 2
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 3
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 4
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 5
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 6
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

processor       : 7
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x1
CPU part        : 0xd07
CPU revision    : 0

~ # cat /proc/zoneinfo | grep Node
Node 0, zone      DMA
Node 0, zone    DMA32
Node 0, zone   Normal
Node 0, zone  Movable
Node 1, zone      DMA
Node 1, zone    DMA32
Node 1, zone   Normal
Node 1, zone  Movable
~ #
```

# 8、在文件系统当中创建新的内容

## 8.1、创建新文件/目录

创建目录

```bash
zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ cd ~/toolchain/busybox-1.36.1/_install/

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1/_install
$ sudo mkdir -p tmp/zdy2

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1/_install
$ ls tmp/
zdy  zdy2

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1/_install
$ 
```

复制Windows中的文件到 这里

```bash
zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ sudo cp /mnt/d/Daniel_Files/linux-6.6/kernel/sched/fair.c _install/tmp/

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ ls _install/tmp/
fair.c  zdy  zdy2

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ 
```

## 8.2、制作新的文件系统镜像

```bash

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1/_install
$ cd ../

zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ sudo ./build_img.sh
32+0 records in
32+0 records out
33554432 bytes (34 MB, 32 MiB) copied, 0.0387027 s, 867 MB/s
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done                            
Creating filesystem with 8192 4k blocks and 8192 inodes

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done


zdy@DESKTOP-3QNUG9S ~/toolchain/busybox-1.36.1
$ 
```

## 8.3、启动QEMU

```bash
// 开启NUMA，两个Node，不调试
sudo qemu-system-aarch64 \
  -machine virt,virtualization=true,gic-version=3 \
  -nographic \
  -m 2G \
  -cpu cortex-a57 \
  -smp cores=8,threads=1 \
  -object memory-backend-ram,id=mem0,size=1G \
  -object memory-backend-ram,id=mem1,size=1G \
  -numa node,memdev=mem0,cpus=0-3,nodeid=0 \
  -numa node,memdev=mem1,cpus=4-7,nodeid=1 \
  -kernel ~/linux_old1/arch/arm64/boot/Image \
  -initrd ~/toolchain/busybox-1.36.1/rootfs.img.gz \
  -append "root=/dev/ram console=ttyAMA0 init=/linuxrc loglevel=8"
```

启动之后，即可看见床剪的目录：

```bash
Please press Enter to activate this console. 
~ # 
~ # ls tmp/
zdy   zdy2
~ # 
```