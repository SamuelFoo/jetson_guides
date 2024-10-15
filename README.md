# Jetson Setup

On first boot, always do a `sudo apt update && sudo apt upgrade`.

## JTOP

If `pip` is not installed, run

```
sudo apt install python3-pip
```

To install `jetson-stats` (the command `jtop`):

```
sudo pip3 install jetson-stats
```

`sudo` is needed for `jetson-stats` to access system information. After installing, reboot the device. After reboot. running `jtop` will give live device information.

# Issues

## 1. Nvidia L4T Kernel Updates on the Seeed Studio J401 Carrier Board

Put `nvidia-l4t-kernel` and related `apt` packages on hold:

```
sudo apt-mark hold nvidia-l4t-kernel
```

Support may change later on, but as of 25 Jun 2024, the L4T kernel on the official nvidia repo (`repo.download.nvidia.com/jetson/common r36.3/main`) does not support the J401.

## 2. USB Modem not recognised as USB Connection

This is due to the Jetpack 6.0 L4T Kernel including the necesary USB network configs / modules. Fix is the recompile the Linux kernel with the neccessary configs.

### Problem

When plugging in a USB modem, the Wired Connection symbol should appear and connection should be through the USB interface. Instead connection to the modem is only possible through wifi, resulting in much lower connection speeds.

### Quick Links

Forum Discussion:
https://forums.developer.nvidia.com/t/jetson-orin-nano-usb-tethering-not-working-jetpack-6/295108

Kernel Customisation Documentation:
https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/SD/Kernel/KernelCustomization.html

Guide to get kernel and filesystem sources and flashing:
https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/IN/QuickStart.html#preparing-a-jetson-developer-kit-for-use

### Diagnosis

To see the linux kernel configs on a running system, check either one of the following:

1. `/proc/config.gz`
2. `/boot/config`
3. `/boot/config-$(uname -r)`

For the Jetson, it will be `/proc/config.gz`. Run the following

```
cat /proc/config.gz | gunzip > running.config
cat running.config | grep CONFIG_USB_NET
```

For "normal" Ubuntu 22.04, it will be `/boot/config-$(uname -r)`. You can `cat` the file directly.

If the configs needed for the modem are set to `n`, the modem would not work.

To see what each config param mean, see https://www.kernelconfig.io/index.html

### Fix

Recompile the kernel with the following configs (**NOTE: This is meant to be an exhaustive list, not a minimal one.** If you care about kernel bloat, feel free to test for the minimally required set of configs.)

```
CONFIG_USB_NET_DRIVERS=m
CONFIG_USB_NET_AX8817X=m
CONFIG_USB_NET_AX88179_178A=m
CONFIG_USB_NET_CDCETHER=m
CONFIG_USB_NET_CDC_EEM=m
CONFIG_USB_NET_CDC_NCM=m
CONFIG_USB_NET_HUAWEI_CDC_NCM=m
CONFIG_USB_NET_CDC_MBIM=m
CONFIG_USB_NET_DM9601=m
CONFIG_USB_NET_SR9700=m
CONFIG_USB_NET_SR9800=m
CONFIG_USB_NET_SMSC75XX=m
CONFIG_USB_NET_SMSC95XX=m
CONFIG_USB_NET_GL620A=m
CONFIG_USB_NET_NET1080=m
CONFIG_USB_NET_PLUSB=m
CONFIG_USB_NET_MCS7830=m
CONFIG_USB_NET_RNDIS_HOST=m
CONFIG_USB_NET_CDC_SUBSET_ENABLE=m
CONFIG_USB_NET_CDC_SUBSET=m
CONFIG_USB_NET_ZAURUS=m
CONFIG_USB_NET_CX82310_ETH=m
CONFIG_USB_NET_KALMIA=m
CONFIG_USB_NET_QMI_WWAN=m
CONFIG_USB_NET_INT51X1=m
CONFIG_USB_NET_CH9200=m
CONFIG_USB_NET_AQC111=m
CONFIG_USB_NET_RNDIS_WLAN=m
CONFIG_USB_NET2272=m
CONFIG_USB_NET2272_DMA=y
CONFIG_USB_NET2280=m
```

#### Download and extract files

You need to download BSP, sample rootfs, kernel source from the Jetson Linux developer page (e.g. https://developer.nvidia.com/embedded/jetson-linux-r363 for Jetson Linux v36.3, Jetpack 6.0)

Three files in total, links for Jetson Linux v36.3, Jetpack 6.0 are pasted below:

https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/jetson_linux_r36.3.0_aarch64.tbz2
https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/tegra_linux_sample-root-filesystem_r36.3.0_aarch64.tbz2
https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/sources/public_sources.tbz2

Download all files in the same directory (e.g. `~/flash_jetson`). This will be the `L4T_INSTALL_PATH`. You can set this as an environment variable, for example:

```
export L4T_INSTALL_PATH=$HOME/flash_jetson
```

Set the following environment variables:

1. `${L4T_RELEASE_PACKAGE}` contains the name of the Jetson Linux release package: `Jetson_Linux_<version>_aarch64.tbz2`.
2. `${SAMPLE_FS_PACKAGE}` contains the name of the sample file system package: `Tegra_Linux_Sample-Root-Filesystem_<version>_aarch64.tbz2`.
3. `${BOARD}` contains the name of a supported configuration of Jetson module and the carrier board. Common values for this field can be found in the Configuration column in the Jetson Modules and Configurations table.

E.g. for flashing Jetpack 6.0 for the Orin NX / Orin Nano developer kit:

```
export L4T_RELEASE_PACKAGE=jetson_linux_r36.3.0_aarch64.tbz2
export SAMPLE_FS_PACKAGE=tegra_linux_sample-root-filesystem_r36.3.0_aarch64.tbz2
export BOARD=jetson-orin-nano-devkit
```

Extract and prepare the binary files:

```
tar xf ${L4T_RELEASE_PACKAGE}
sudo tar xpf ${SAMPLE_FS_PACKAGE} -C Linux_for_Tegra/rootfs/
cd Linux_for_Tegra/
sudo ./tools/l4t_flash_prerequisites.sh
sudo ./apply_binaries.sh
```

Extract the kernel source:

```
tar xf public_sources.tbz2 -C ${L4T_INSTALL_PATH}/Linux_for_Tegra/..
cd ${L4T_INSTALL_PATH}/Linux_for_Tegra/source
tar xf kernel_src.tbz2
tar xf kernel_oot_modules_src.tbz2
tar xf nvidia_kernel_display_driver_source.tbz2
```

**Take note of `..` after `Linux_for_Tegra/`** in the first line.

#### Install Pre-requisites

1. Git

```
sudo apt install git-core
```

Your system must have the default Git port 9418 open for outbound connections.

2. Build Utilities

```
sudo apt install build-essential bc
```

3. Jetson Linux Toolchain

https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/AT/JetsonLinuxToolchain.html#at-jetsonlinuxtoolchain

4. Other C++ Tools

These are supposed to have been installed above, but if they are not, run:

```
sudo apt install flex bison
```

#### Change Kernel Configuration

Go to the directory `Linux_for_Tegra/source/kernel/kernel-jammy-src/`

Run:

```
make ARCH=arm64 defconfig
make ARCH=arm64 menuconfig
```

At this point, the Kernel Configuration Menu will appear. To see which settings need to be adjusted, see https://www.kernelconfig.io/index.html or press `/` to search.

After editing the Kernel Configuration, make sure to save the changes by running (in the same directory):

```
make ARCH=arm64 savedefconfig
cp ./defconfig ./arch/arm64/configs
```

#### Build Kernel, OOT and DTBs

**NOTE: if at any time you change the terminal instance, re-set the environment variables below**

```
export L4T_INSTALL_PATH=$HOME/flash_jetson
export L4T_RELEASE_PACKAGE=jetson_linux_r36.3.0_aarch64.tbz2
export SAMPLE_FS_PACKAGE=tegra_linux_sample-root-filesystem_r36.3.0_aarch64.tbz2
export BOARD=jetson-orin-nano-devkit
export CROSS_COMPILE=$HOME/l4t-gcc/aarch64--glibc--stable-2022.08-1/bin/aarch64-buildroot-linux-gnu-
```

1. Build the Jetson Linux Kernel

```
cd ${L4T_INSTALL_PATH}/Linux_for_Tegra/source

./generic_rt_build.sh "enable"

make -C kernel

export INSTALL_MOD_PATH=${L4T_INSTALL_PATH}/Linux_for_Tegra/rootfs/
sudo -E make install -C kernel
cp kernel/kernel-jammy-src/arch/arm64/boot/Image \
  ${L4T_INSTALL_PATH}/Linux_for_Tegra/kernel/Image
```

2. Build the NVIDIA Out-of-Tree Modules

```
cd ${L4T_INSTALL_PATH}/Linux_for_Tegra/source

export IGNORE_PREEMPT_RT_PRESENCE=1
export KERNEL_HEADERS=$PWD/kernel/kernel-jammy-src
make modules

export INSTALL_MOD_PATH=${L4T_INSTALL_PATH}/Linux_for_Tegra/rootfs/
sudo -E make modules_install

cd ${L4T_INSTALL_PATH}/Linux_for_Tegra
sudo ./tools/l4t_update_initrd.sh
```

3. Build the Device Tree Blobs

```
cd ${L4T_INSTALL_PATH}/Linux_for_Tegra/source

export KERNEL_HEADERS=$PWD/kernel/kernel-jammy-src
make dtbs

cp nvidia-oot/device-tree/platform/generic-dts/dtbs/* \
  ${L4T_INSTALL_PATH}/Linux_for_Tegra/kernel/dtb/
```

#### Flash the Developer Kit

https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/IN/QuickStart.html#to-flash-the-jetson-developer-kit-operating-software

Start from step 3.

The flash may fail once or twice. Simply power off the device, unplug the power and the USB-C connection, plug back the power, turn the device on, reconnect the USB-C and then try to flash again. To avoid complications, please follow the steps **in order**.

If you see the error "No such device: /sys/class/net/usb0", or the flash failing due to the host laptop being unable to establish a SSH connection to the Jetson (e.g. the flash being stuck at "Waiting for device to expose ssh..."), **terminate all network-related processes and applications**. This can include, but are not limited to, tailscale, Wifi, radio antennas etc. One can do this by doing `sudo nmcli radio wifi off` (to revert this **after the flash**, do `sudo nmcli radio wifi on`) and `sudo ifconfig <network_name> down` (to revert this, do `sudo ifconfig <network_name> up`).

## 3. USB WiFi dongle not recognised

Same as [## 2. USB Modem not recognised as USB Connection](##-2.-USB-Modem-not-recognised-as-USB-Connection), we recompile the kernel with device-specific configs.

The following were used for *TP-Link TL-WN727N*:

```
<Insert configs here>
```

Furthermore, as of 15 Oct 2024, the `rtl8xxxu` kernel module bundled with Jetson Linux 36.3 does not contain mappings for some devices. To check, run `lsusb` and note the vendor and product IDs of your device. For example, in

```
<Insert lsusb output here>
```

`2357` is the vendor ID and `0101c` is the product ID. 

Check that `Linux_for_Tegra/source/kernel/kernel-jammy-src/drivers/net/wireless/realtek/rtl8xxxu/rtl8xxxu_core.c` has an interface defined for your device under `static const struct usb_device_id dev_table[]`. If an interface is not defined for your device, you will need to recompile the kernel module. Clone the repository https://github.com/SamuelFoo/rtl8xxxu and replace `Linux_for_Tegra/source/kernel/kernel-jammy-src/drivers/net/wireless/realtek/rtl8xxxu`. Remove the `.git` folder and then continue from [#### Build Kernel, OOT and DTBs](####-Build-Kernel,-OOT-and-DTBs). (You should build the kernel module using the Jetson Linux toolchain, not your host computer's `make`.)
