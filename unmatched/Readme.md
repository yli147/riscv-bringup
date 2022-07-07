# Building the SiFive Unmatched RISC-V board boot requirements <!-- omit in toc -->

The objective of this guide is to provide an full solution on building the necessary packages to boot the SiFive Unmatched RISC-V board boot requirements.

There is also a SiFive Unmatched, prebuilt SDcard image at [https://github.com/carlosedp/riscv-bringup/releases/download/v1.0/UbuntuHippo-RISC-V.img.gz](https://github.com/carlosedp/riscv-bringup/releases/download/v1.0/UbuntuHippo-RISC-V.img.gz).

This is still a moving target so the process might change in the future. I confirm that with used versions everything works. **Please keep the versions as stated since patches baselines changes as revisions evolve.**

The stack is composed on U-Boot SPL, OpenSBI (the second-stage boot loader) loading U-Boot and this present the Linux Kernel options (from `extlinux.conf`) allowing one to add and change Kernel versions.

Below is a diagram of the process:

```sh

                                    +----------------------------------+      Extlinux.conf                Linux
                                    |                                  |
                                    |         SBL - U-Boot ITB         |    +----------------+    +--------------------+
                                    |                                  |    |                |    |                    |
+--------------+   +----------+     |  +-----------+    +-----------+  |    | Unleashed Menu |    | Starting kernel ...|
|              |   |          |     |  |           |    |           |  |    |                |    | [0.00] Linux versio|
|  ROM - ZSBL  |   |  U-Boot  |     |  |           |    |           |  |    | 1. Kernel 5.13 |    | [0.00] Kernel comma|
|  In the SoC  +-->+  SPL     +---->+  |  OpenSBI  +--->+  U-Boot + |  +--->+ 2. Kernel 5.x  +--->+ ..                 |
|              |   |          |     |  |           |    |  DTB      |  |    |                |    | ...                |
+--------------+   +----------+     |  |           |    |           |  |    |                |    |                    |
                                    |  |           |    |           |  |    |                |    |                    |
                                    |  +-----------+    +-----------+  |    +----------------+    +--------------------+
                                    |                                  |
                                    +----------------------------------+

```

* **ZSBL** - Zero Stage Bootloader - Code in the ROM of the board
* **FSBL** - First Stage Bootloader - U-Boot SPL - Loader that is called from ROM.
* **SBL** -  Second Bootloader - OpenSBI - Supervisor Binary Interface. [Source](https://github.com/riscv/opensbi/)
* **U-Boot** - Universal Boot Loader. [Docs](https://www.denx.de/wiki/U-Boot)
* **Extlinux** - Syslinux compatible configuration to load Linux Kernel and DTB thru a configurable menu from a filesystem.

## Table of Contents <!-- omit in toc -->

* [Install Toolchain to build Kernel](#install-toolchain-to-build-kernel)
* [Clone repositories](#clone-repositories)
* [Build OpenSBI](#build-opensbi)
* [Build U-Boot](#build-u-boot)
* [Linux Kernel](#linux-kernel)
  * [Kernel 5.13 checkout and patches](#kernel-513-checkout-and-patches)
  * [Building the Kernel](#building-the-kernel)
* [Building or getting a root filesystem](#building-or-getting-a-root-filesystem)
* [Creating an SDCard Image file](#creating-an-sdcard-image-file)
* [MSEL for Unmatched](#msel-for-unmatched)
* [Use NVME as root filesystem](#use-nvme-as-root-filesystem)
* [Installing/Updating new Kernel and Bootloader packages](#installingupdating-new-kernel-and-bootloader-packages)
* [Post configs](#post-configs)
* [References](#references)

## Install Toolchain to build Kernel

This process has been done in a x86_64 VM running Ubuntu Focal Fossa.

First install the RISC-V toolchain. You can build from source by using the commands below or download a pre-built one from [Bootlin](https://toolchains.bootlin.com/releases_riscv64.html).

```sh
git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
sudo apt-get install -y autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev libncurses-dev device-tree-compiler libssl-dev gdisk swig
./configure --prefix=/opt/riscv
sudo make linux -j6
export PATH=/opt/riscv/bin:$PATH
echo "export PATH=/opt/riscv/bin:$PATH" >> ~/.bashrc
```

Adjust the triplet in the `CROSS_COMPILE` according to the used toolchain. The one from Bootlin is ``

## Clone repositories

Clone the required repositories. You need OpenSBI (Second stage bootloader), U-Boot and the Linux kernel. I keep all in one directory.

```sh
# mkdir unmatched
git clone https://github.com/yli147/riscv-bringup.git
cd riscv-bringup/unmatched

# OpenSBI
git clone https://github.com/riscv/opensbi

# U-Boot
git clone https://source.denx.de/u-boot/u-boot.git

# Linux Kernel (Stable)
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git

# SiFive patches
git clone https://github.com/sifive/meta-sifive

# Checkout meta-sifive patches
pushd meta-sifive
lasttag=2022.02
echo $lasttag
git checkout $lasttag
popd
```

## Build OpenSBI

OpenSBI is the interface between the firmware and the bootloader.

We apply Unmatched patches until they get upstreamed

```sh
pushd opensbi
# Checkout the working version
git checkout 4998a712b2ab504eff306110879ee05af6050177

# Build
make CROSS_COMPILE=riscv64-unknown-linux-gnu- PLATFORM=generic INSTALL_LIB_PATH=lib FW_PIC=n

# Export OpenSBI dynamic firmware to be used by U-Boot
export OPENSBI=`realpath build/platform/generic/firmware/fw_dynamic.bin`

popd
```

This will generate the file `build/platform/generic/firmware/fw_dynamic.bin` that will be used by U-Boot.

## Build U-Boot

U-Boot is the bootloader used to load the Kernel from the filesystem. It has a menu that allows one to select which version of the Kernel to use (if needed).

We use latest released version with Unmatched patches until they get upstreamed.

```bash
pushd u-boot
git checkout d637294e264adfeb29f390dfc393106fd4d41b17

for f in ../meta-sifive/recipes-bsp/u-boot/files/riscv64/*.patch; do echo $f;patch -p1 < $f;done

# To change clock rate, edit file (at own risk)
vi arch/riscv/dts/fu740-c000-u-boot.dtsi
# clock is at `assigned-clock-rates` key, my board works at 1.5Ghz

# Check if OpenSBI is exported
ls -ltr $OPENSBI

CROSS_COMPILE=riscv64-linux-gnu- make sifive_unmatched_defconfig
CROSS_COMPILE=riscv64-linux-gnu- make menuconfig # if needed
CROSS_COMPILE=riscv64-linux-gnu- make -j`nproc`
popd
```

This will generate the file `u-boot.itb` and `spl/u-boot-spl.bin` to be flashed to the SDcard or image.

## Linux Kernel

### Kernel 5.13 checkout and patches

The patches supporting the Unmatched targets the 5.13 Kernel:

```sh
pushd linux
git checkout linux-5.13.19
```

Apply Unmatched patches until they get upstream and rebase on latest 5.13:

```sh
for f in ../meta-sifive/recipes-kernel/linux/files/*.patch; do echo $f;patch -p1 < $f;done
# On some Unmatched boards, the DA9063 PMIC is a newer revision not yet supported by upstream Kernel
patch -p1 < ../patches/0001-mfd-da9063-Add-support-for-latest-EA-silicon-revisio.patch
git rebase origin/linux-5.13.y
```

Apply extra patches supporting the RTC (Realtime Clock) and Reboot:

```sh
patch -p1 < ../patches/kernel-rtc.diff
```

### Building the Kernel

Apply the defconfig supporting Unmatched. This config has most requirements for containers and networking features built-in and is confirmed to work.

```sh
cp ../patches/linux-5.13-defconfig ./.config
```

If you prefer to have a Kernel without the `-dirty` naming due to applied patches, do:

```sh
touch .scmversion
```

Build the kernel. The `menuconfig` line is in case one want to customize any parameter (adjust the CROSS_COMPILE triplet if using a different toolchain).

```sh
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv olddefconfig
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv -j`nproc`

# Package kernel and modules into a tarball and debian package
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv INSTALL_MOD_STRIP=1 -j`nproc` tarbz2-pkg
make CROSS_COMPILE=riscv64-linux-gnu- ARCH=riscv INSTALL_MOD_STRIP=1 -j`nproc` bindeb-pkg
$$
# Build version string
version=`cat include/config/kernel.release`
echo $version
popd
```

Check if building produced the files `linux/arch/riscv/boot/Image` and `linux/arch/riscv/boot/dts/sifive/hifive-unmatched-a00.dtb`. This is the kernel file and the dtb that is the descriptor for the board hardware.

The last command will create three `.deb` packages in parent directory. They are `linux-headers-5.12.19...`, `linux-libc-dev_5.12.19...` and `linux-image-5.12.19...` where the ellipsis denote de dirty version. They will be used on the rootfs to install the Kernel and modules.

## Building or getting a root filesystem

As the root filesystem, you can choose between downloading a pre-built Debian or Ubuntu or build the rootfs yourself.

The pre-built Ubuntu Hippo tarball can be downloaded with: `wget -O rootfs.tar.bz2 https://github.com/yli147/riscv-bringup/releases/download/v1.0/UbuntuFocal-riscv64-rootfs.tar.gz`.

The pre-built Debian tarball can be downloaded with: `wget -O rootfs.tar.bz2 https://github.com/yli147/riscv-bringup/releases/download/v1.0/debian-sid-riscv64-rootfs-20200108.tar.bz2`.

If you want to build a Debian rootfs from scratch, [check this guide](https://github.com/yli147/riscv-bringup/blob/master/Debian-Rootfs-Guide.md).

If you want to build an Ubuntu rootfs from scratch, [check this guide](https://github.com/yli147/riscv-bringup/blob/master/Ubuntu-Rootfs-Guide.md).

## Creating an SDCard Image file

In this option, the `/boot` dir, where the Kernel is stored is in the same partition as the rootfs.

The bootloader and root filesystem runs from the SDcard. I recommend using an NVME drive for the rootfs which is significantly faster than the SDcard. In this case, the bootloader stays in the SDcard which should be in the board as well.

If using an NVME drive, check the section [using NVME as root filesystem](#use-nvme-as-root-filesystem) for instructions on how to move the data to the drive and adjust boot parameters.

```sh
# Create a 1.5GB image
dd if=/dev/zero of=UbuntuHippo-RISC-V.img bs=1M count=1536

# Partition image with correct disk IDs
sudo sgdisk -g --clear --set-alignment=1 \
       --new=1:34:+1M:    --change-name=1:'u-boot-spl'    --typecode=1:5b193300-fc78-40cd-8002-e86c45580b47 \
       --new=2:2082:+4M:  --change-name=2:'opensbi-uboot' --typecode=2:2e54b353-1271-4842-806f-e436d6af6985 \
       --new=3:16384:-0   --change-name=3:'rootfs'        --attributes=3:set:2  \
       UbuntuHippo-RISC-V.img

# Mount image in loop device
sudo losetup --partscan --find --show UbuntuHippo-RISC-V.img

# Write the bootloader partitions. Adjust "loop0" to your loop device created in previous command if needed.
sudo dd if=u-boot/spl/u-boot-spl.bin of=/dev/loop0p1 bs=4k iflag=fullblock oflag=direct conv=fsync status=progress
sudo dd if=u-boot/u-boot.itb of=/dev/loop0p2 bs=4k iflag=fullblock oflag=direct conv=fsync status=progress

# Create and mount root filesystem
sudo mkfs.ext4 /dev/loop0p3
sudo e2label /dev/loop0p3 rootfs
sudo mount /dev/loop0p3 /mnt

# Unpack root filesystem
sudo tar vxf Ubuntu-Hippo-rootfs.tar.gz -C /mnt --strip-components=1

# Copy Linux Kernel packages to the rootfs
sudo cp linux-*.deb /mnt/tmp

# Chroot to the image partition to install Kernel
sudo chroot /mnt bin/bash
cd /tmp

# Install Linux kernel and modules
apt install -y ./*.deb
rm *.deb

# Exit chroot
exit

# Copy DTBs. Check if $version is set, otherwise do
version=`cat linux/include/config/kernel.release`
echo $version
sudo mkdir -p /mnt/boot/dtbs/
sudo cp -R /mnt/usr/lib/linux-image-$version/ /mnt/boot/dtbs/$version

# Create extlinux.conf file for U-Boot
sudo mkdir -p /mnt/boot/extlinux
cat << EOF | sudo tee /mnt/boot/extlinux/extlinux.conf
menu title SiFive Unmatched Boot Options
timeout 100
default kernel-$version

label kernel-$version
        menu label Linux kernel-$version
        kernel /boot/vmlinuz-$version
        fdt /boot/dtbs/$version/sifive/hifive-unmatched-a00.dtb
        initrd /boot/initrd.img-$version
        append earlyprintk rw root=/dev/mmcblk0p3 rootfstype=ext4 rootwait console=ttySIF0,115200 LANG=en_US.UTF-8 earlycon

label recovery-kernel-$version
        menu label Linux kernel-$version (recovery mode)
        kernel /boot/vmlinuz-$version
        fdt /boot/dtbs/$version/sifive/hifive-unmatched-a00.dtb
        initrd /boot/initrd.img-$version
        append earlyprintk rw root=/dev/mmcblk0p3 rootfstype=ext4 rootwait console=ttySIF0,115200 LANG=en_US.UTF-8 earlycon single
EOF

# Unmount image
sudo umount /mnt
sudo losetup -d /dev/loop0

pushd u-boot
make qemu-riscv64_smode_defconfig
make CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)
popd

qemu-system-riscv64 \
-machine virt -nographic -m 2048 -smp 4 \
-bios opensbi/build/platform/generic/firmware/fw_jump.elf \
-kernel u-boot/u-boot.bin \
-device virtio-net-device,netdev=eth0 -netdev user,id=eth0 \
-drive file=UbuntuHippo-RISC-V.img,format=raw,if=virtio
```

Flash to SDCard and resize the root partition

```sh
sudo dd if=UbuntuHippo-RISC-V.img of=/dev/sdc bs=64k iflag=fullblock oflag=direct conv=fsync status=progress

echo "- +" | sfdisk -N 3 /dev/sdc
sudo resize2fs /dev/sdc3
```

Insert the SDcard in the Unmatched board and boot. Follow the output in serial console via USB.

The root password for the rootfs is *riscv* and root login thru SSH is **enabled**. The network interface will get an IP from DHCP.

## MSEL for Unmatched

By default MSEL on Unmatched is set to use the SDcard. Below is the default configuration for DIP switches (located next to Assembly Number and RTC battery). Check if it's correct on your board.

```sh
  +----------> CHIPIDSEL
  | +--------> MSEL3
  | | +------> MSEL2
  | | | +----> MSEL1
  | | | | +--> MSEL0
  | | | | |
 +-+-+-+-+-+
 | |X| |X|X| ON(1)          /|\
 | | | | | |                 |  Edge of the PCB
 |X| |X| | | OFF(0)          |
 +-+-+-+-+-+                 |
BOOT MODE SEL
```

## Use NVME as root filesystem

To use an NVME as the main drive, follow these steps. This is done on the Unmatched itself booted from the SDcard.

Add NVME module to initramfs and update it:

```sh
echo "nvme" >> /etc/initramfs-tools/modules
sudo update-initramfs -u -k all
```

Partition the drive and create the filesystems. If desired, create a swap partition.

```sh
# Create two partitions, one for rootfs and another for swap (8GB is enough)
# Keep the swap partition at the end
sudo fdisk  /dev/nvme0n1

# create partitions and save

sudo mkfs.ext4 /dev/nvme0n1p1
sudo mkswap -L swap1 /dev/nvme0n1p2
```

Copy the root partition from the SDcard to the NVME drive:

```sh
sudo mount /dev/nvme0n1p1 /mnt
sudo rsync -axHAWX --numeric-ids --info=progress2 --exclude={"/dev/","/proc/","/sys/","/tmp/","/run/","/mnt/","/media/*","/lost+found"} / /mnt
sudo mkdir /mnt/{dev,proc,sys,tmp,run,mnt}
sudo sync

sudo e2label /dev/nvme0n1p1 rootfsnvme
# Rename mount label on /etc/fstab
sudo sed -i "s/.*rootfs.*/LABEL=rootfsnvme      \/      ext4    user_xattr,errors=remount-ro    0       1/g" /mnt/etc/fstab

echo "/dev/nvme0n1p2		swap	swap	defaults			0 0" |sudo tee -a /mnt/etc/fstab
sudo umount /mnt
```

Edit `/boot/extlinux/extlinux.conf` to point the root partition to the NVME. Change `root=/dev/mmcblk0p3` to `root=/dev/nvme0n1p1` on both Kernel instances (in `append` lines).

Now reboot and check if U-boot loaded the Kernel and rootfs from NVME.

## Installing/Updating new Kernel and Bootloader packages

To build new Kernel packages and U-boot/OpenSBI binaries, follow the respective sections above.

Transfer the `u-boot-spl.bin` and `u-boot.itb` files and the three kernel `.deb` files to the Unmatched.

```sh
# Write the new U-boot and SPL to the SDcard
sudo dd if=u-boot.itb of=/dev/mmcblk0p2 bs=4k oflag=direct
sudo dd if=u-boot-spl.bin of=/dev/mmcblk0p1 bs=4k oflag=direct

#Install Kernel packages (set the version variable based on your built kernel)
version=5.13.19
sudo apt install ./*.deb

# Copy DTBs
sudo cp -R /usr/lib/linux-image-$version/ /boot/dtbs/$version

cat << EOF | sudo tee -a /boot/extlinux/extlinux.conf

label kernel-$version
        menu label Linux kernel-$version
        kernel /boot/vmlinuz-$version
        fdt /boot/dtbs/$version/sifive/hifive-unmatched-a00.dtb
        initrd /boot/initrd.img-$version
        append earlyprintk rw root=$(mount |grep "on /\ "|cut -d" " -f1) rootfstype=ext4 rootwait console=ttySIF0,115200 LANG=en_US.UTF-8 earlycon

label recovery-kernel-$version
        menu label Linux kernel-$version (recovery mode)
        kernel /boot/vmlinuz-$version
        fdt /boot/dtbs/$version/sifive/hifive-unmatched-a00.dtb
        initrd /boot/initrd.img-$version
        append earlyprintk rw root=$(mount |grep "on /\ "|cut -d" " -f1) rootfstype=ext4 rootwait console=ttySIF0,115200 LANG=en_US.UTF-8 earlycon single
EOF
```

Edit `/boot/extlinux/extlinux.conf` to set your default Kernel editing the line `default` pointing to the desired `label`.

```sh
sudo sed s/default.*/"default kernel-$version"/ -i /boot/extlinux/extlinux.conf
```

Reboot

## Post configs

Create heartbeat led udev rule:

```sh
cat << EOF | sudo tee /etc/udev/rules.d/99-pwm-leds.rules
# D12 LED: heartbeat
SUBSYSTEM=="leds", KERNEL=="green:d12", ACTION=="add", ATTR{trigger}="heartbeat"

# D12 RGB LED: boot status
SUBSYSTEM=="leds", KERNEL=="green:d2", ACTION=="add", ATTR{trigger}="default-on", ATTR{brightness}="25"
SUBSYSTEM=="leds", KERNEL=="red:d2", ACTION=="add", ATTR{trigger}="default-on", ATTR{brightness}="25"
SUBSYSTEM=="leds", KERNEL=="blue:d2", ACTION=="add", ATTR{trigger}="default-on", ATTR{brightness}="25"
EOF


cat << EOF | sudo tee /etc/systemd/system/led-bootstate-green.service
[Unit]
Description="Change the color of D2 LED to green"
ConditionPathIsReadWrite=/sys/class/leds/green:d2/trigger
ConditionPathIsReadWrite=/sys/class/leds/red:d2/trigger
ConditionPathIsReadWrite=/sys/class/leds/blue:d2/trigger

[Service]
Type=oneshot
ExecStart=/bin/bash -c '/bin/echo "default-on" > /sys/class/leds/green:d2/trigger'
ExecStart=/bin/bash -c '/bin/echo "25" > /sys/class/leds/green:d2/brightness'
ExecStart=/bin/bash -c '/bin/echo "none" > /sys/class/leds/red:d2/trigger'
ExecStart=/bin/bash -c '/bin/echo "none" > /sys/class/leds/blue:d2/trigger'
EOF

cat << EOF | sudo tee /etc/systemd/system/led-bootstate-green.timer
[Unit]
Description=timer to trun-on green LED
After=getty.target

[Timer]
OnActiveSec=3min
Unit=led-bootstate-green.service

[Install]
WantedBy=timers.target
EOF

cat << EOF | sudo tee -a /etc/sensors3.conf

chip "tmp451-*"

    label temp1 "M/B Temp"
    label temp2 "CPU Temp"
EOF

sudo systemctl daemon-reload
sudo systemctl enable led-bootstate-green
sudo systemctl start led-bootstate-green
```

## References

* HiFive Unleashed OpenSBI - <https://github.com/riscv/opensbi/blob/master/docs/platform/sifive_fu540.md>
* HiFive Unleashed U-Boot - <https://gitlab.denx.de/u-boot/u-boot/blob/master/doc/board/sifive/fu540.rst>
* OpenSBI Deep Dive - <https://content.riscv.org/wp-content/uploads/2019/06/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf>
* The future of Supervisor Binary Interface(SBI) - <https://www.youtube.com/watch?v=d50mzglm2jU>
