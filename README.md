# Tutorial：RISC-V环境下的QEMU/KVM虚拟化

<!--ts-->
<!--te-->

## 准备交叉编译工具链

从源码编译：
https://github.com/riscv-collab/riscv-gnu-toolchain

从APT安装（Ubuntu 20.04或更新）：
`sudo apt-get install crossbuild-essential-riscv64`

## 基于RISC-V环境运行Host Linux

本tutorial会介绍三种不同的RISC-V模拟环境的搭建：

### 1. QEMU模拟环境

- 优势：不依赖RISC-V硬件，开发快速便捷

- 缺点：与真实硬件存在较大差异，未支持模拟微体系结构（如cache、TLB）等

适用场景：软件功能开发、验证、调试等

#### 构建支持RISC-V虚拟化扩展（H-extension）的QEMU

安装依赖：https://wiki.qemu.org/Hosts/Linux

```bash
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build
```

从源码编译QEMU：
> 注：QEMU版本需要7.0或更新

```bash
git clone https://gitlab.com/qemu-project/qemu.git rv-emul-qemu
cd rv-emul-qemu
git submodule init
git submodule update --recursive
./configure --target-list="riscv32-softmmu riscv64-softmmu"
make
cd ..
```

#### 构建Host Linux

> 注：Linux版本需要5.16或更新，config可自行调整，需保证CONFIG_KVM开启

```bash
git clone https://github.com/kvm-riscv/linux.git
export ARCH=riscv
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
mkdir build-host
make -C linux O=`pwd`/build-host defconfig
make -C linux O=`pwd`/build-host
```

#### 构建Host RootFS

生成`rootfs_kvm_host.img`

```bash
export ARCH=riscv
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
git clone https://github.com/kvm-riscv/howto.git
wget https://busybox.net/downloads/busybox-1.33.1.tar.bz2
tar -C . -xvf ./busybox-1.33.1.tar.bz2
mv ./busybox-1.33.1 ./busybox-1.33.1-kvm-host
cp -f ./howto/configs/busybox-1.33.1_defconfig busybox-1.33.1-kvm-host/.config
make -C busybox-1.33.1-kvm-host oldconfig
make -C busybox-1.33.1-kvm-host install
mkdir -p busybox-1.33.1-kvm-host/_install/etc/init.d
mkdir -p busybox-1.33.1-kvm-host/_install/dev
mkdir -p busybox-1.33.1-kvm-host/_install/proc
mkdir -p busybox-1.33.1-kvm-host/_install/sys
mkdir -p busybox-1.33.1-kvm-host/_install/apps
ln -sf /sbin/init busybox-1.33.1-kvm-host/_install/init
cp -f ./howto/configs/busybox/fstab busybox-1.33.1-kvm-host/_install/etc/fstab
cp -f ./howto/configs/busybox/rcS busybox-1.33.1-kvm-host/_install/etc/init.d/rcS
cp -f ./howto/configs/busybox/motd busybox-1.33.1-kvm-host/_install/etc/motd
cp -f ./build-host/arch/riscv/kvm/kvm.ko busybox-1.33.1-kvm-host/_install/apps
cd busybox-1.33.1-kvm-host/_install; find ./ | cpio -o -H newc > ../../rootfs_kvm_host.img; cd -
```

#### 基于QEMU运行Host Linux

```bash
./rv-emul-qemu/build/qemu-system-riscv64 \
    -cpu rv64 \
    -M virt \
    -m 1024M \
    -nographic \
    -kernel ./build-host/arch/riscv/boot/Image \
    -initrd ./rootfs_kvm_host.img \
    -append "root=/dev/ram rw console=ttyS0 earlycon=sbi"
```

启动后验证Host Linux的KVM是否使能成功

```
           _  _
          | ||_|
          | | _ ____  _   _  _  _ 
          | || |  _ \| | | |\ \/ /
          | || | | | | |_| |/    \
          |_||_|_| |_|\____|\_/\_/

               Busybox Rootfs

Please press Enter to activate this console. 
/ # 
/ # cat /proc/cpuinfo 
processor	: 0
hart		: 0
isa		: rv64imafdcsuh
mmu		: sv48

/ # cat /proc/interrupts 
           CPU0       
  1:          0  SiFive PLIC  11  101000.rtc
  2:        153  SiFive PLIC  10  ttyS0
  5:        449  RISC-V INTC   5  riscv-timer
IPI0:         0  Rescheduling interrupts
IPI1:         0  Function call interrupts
IPI2:         0  CPU stop interrupts

/ # insmod apps/kvm.ko
/ # dmesg | grep kvm
[    1.094977] kvm [1]: hypervisor extension available
[    1.096555] kvm [1]: host has 14 VMID bits
[   20.254215] random: fast init done
```

---

### 2. SiFive U500 FPGA模拟环境

- 优势：硬件仿真，支持模拟微体系结构（如cache、TLB）等

- 缺点：运行速度较慢，与真实芯片存在一定差异因此性能不准确

适用场景：硬件功能开发、验证、调试等

环境搭建请参考FPGA官方文档：https://static.dev.sifive.com/SiFive-U500-vc707-gettingstarted-v0.2.pdf

---

### 3. FireSim模拟环境

- 优势：Cycle-Accurate的硬件仿真

- 缺点：运行速度极慢，价格高昂

适用场景：硬件性能测试等

环境搭建请参考：https://github.com/kvm-riscv/howto/wiki/KVM-RISC-V-64bit-on-FireSim#using-the-firesim-to-load-the-software-image

---

## 使用KVM运行Guest Linux

在上述三种RISC-V环境中，基于Host Linux使用KVM的步骤基本一致，故以QEMU模拟环境为例统一介绍如下：

### 构建Guest Linux

> 注：Linux版本与config可自行调整，如需使用I/O则开启CONFIG_VIRTIO相关选项

```bash
git clone https://github.com/kvm-riscv/linux.git
export ARCH=riscv
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
mkdir build-guest
make -C linux O=`pwd`/build-guest defconfig
make -C linux O=`pwd`/build-guest
```

### 构建用于KVM的QEMU

```bash
git clone https://gitlab.com/qemu-project/qemu.git kvm-qemu
cd kvm-qemu
git submodule init
git submodule update --recursive
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu- 
./configure --target-list="riscv32-softmmu riscv64-softmmu"
make
cd ..
```

### 构建Guest RootFS

生成`rootfs_kvm_guest.img`

```bash
export ARCH=riscv
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
tar -C . -xvf ./busybox-1.33.1.tar.bz2
mv ./busybox-1.33.1 ./busybox-1.33.1-kvm-guest
cp -f ./howto/configs/busybox-1.33.1_defconfig busybox-1.33.1-kvm-guest/.config
make -C busybox-1.33.1-kvm-guest oldconfig
make -C busybox-1.33.1-kvm-guest install
mkdir -p busybox-1.33.1-kvm-guest/_install/etc/init.d
mkdir -p busybox-1.33.1-kvm-guest/_install/dev
mkdir -p busybox-1.33.1-kvm-guest/_install/proc
mkdir -p busybox-1.33.1-kvm-guest/_install/sys
mkdir -p busybox-1.33.1-kvm-guest/_install/apps
ln -sf /sbin/init busybox-1.33.1-kvm-guest/_install/init
cp -f ./howto/configs/busybox/fstab busybox-1.33.1-kvm-guest/_install/etc/fstab
cp -f ./howto/configs/busybox/rcS busybox-1.33.1-kvm-guest/_install/etc/init.d/rcS
cp -f ./howto/configs/busybox/motd busybox-1.33.1-kvm-guest/_install/etc/motd
cd busybox-1.33.1-kvm-guest/_install; find ./ | cpio -o -H newc > ../../rootfs_kvm_guest.img; cd -
```

### 将Guest Linux需要的QEMU与RootFS镜像放入Host RootFS

重新生成`rootfs_kvm_host.img`

```bash
cp -f ./build-guest/arch/riscv/boot/Image busybox-1.33.1-kvm-host/_install/apps
cp -f ./kvm-qemu/build/qemu-system-riscv64 busybox-1.33.1-kvm-host/_install/apps
cp -f ./rootfs_kvm_guest.img busybox-1.33.1-kvm-host/_install/apps
cd busybox-1.33.1-kvm-host/_install; find ./ | cpio -o -H newc > ../../rootfs_kvm_host.img; cd -
```

### 使用新Host RootFS运行Host Linux

```bash
./rv-emul-qemu/build/qemu-system-riscv64 \
    -cpu rv64 \
    -M virt \
    -m 1024M \
    -nographic \
    -kernel ./build-host/arch/riscv/boot/Image \
    -initrd ./rootfs_kvm_host.img \
    -append "root=/dev/ram rw console=ttyS0 earlycon=sbi"
```

### 基于KVM运行Guest Linux

```bash
./qemu-system-riscv64 \
    -cpu host \
    --enable-kvm \
    -M virt \
    -m 512M \
    -nographic \
    -kernel ./Image \
    -initrd ./rootfs_kvm_guest.img \
    -append "root=/dev/ram rw console=ttyS0 earlycon=sbi"
```
