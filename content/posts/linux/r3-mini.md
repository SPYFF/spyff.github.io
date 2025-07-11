---
categories: ["linux", "router"]
date: 2025-07-05T07:00:00Z
summary: Installing stock Debian into Banana Pi BPI-R3 Mini router board with u-boot chainloading
title: BPI-R3 Mini with stock Debian
slug: "r3-mini"
#canonical: "https://fejes.dev/posts/net/so-priority-cmsg/"
draft: true
---

# Intro

Banana Pi BPI-R3 Mini is a palm sized router/development board.
It's quite powerful with Mediatek MT7986 quad-core CPU and 2Gb RAM.
With 2x2.5GbE and 2.4/5GHz 802.11ax Wi-Fi good candidate for a portable router.
However in this post I would like to go beyond simply flashing an OpenWRT into it
and call it a day.

My main goal is understand how it boots then install stock Debian into it.
By stock I mean the installer officially provided by the Debian project,
no modifications or custom built debootstrap images.
Ideally, I want to keep the OpenWRT system as well (just in case),
and install Debian into the NVMe SSD and booting it as the default system.
Yes, despite the small size it has two M.2 slot, and one of them is PCIe
and the other is USB, so one can equip the board with an NVMe SSD and 5G modem.

## Available resources

The great thing about this board is the plenty of available
official and community maintained documentations.
Also, it has official and unofficial BSPs (Board Support Package,
essentially customized Linux kernel and rootfs tailored to the board).
If one don't want to bother with tweaking described in the following,
it's very easy to get the board up and running with these.

## About the BPI-R3 Mini

As someone might guess from the name mini, there is a conventional
router form factor version of the board.
This packs more Ethernet ports, even two 10GbE.
What is more interesting, it has SD card slot which makes the experimenting
much more convenient: if something wrong with our kernel config or
rootfs, simply unplug the SD card, flash the new image into it and retry.

Unfortunately the R3 mini does not have SD card slot.
It has an eMMC and SPI flash soldered on the board and that's all.
This is not a huge problem, but we have to be a bit more careful,
if something go wrong, we can end up soft-bricked device which takes
some effort to recover (see later).

We will need an USB flash drive and USB-Ethernet dongle for
our experiments.
These USB-Ethernet dongles are most commonly use Realtek 8152/8153/8156 or
ASIX AX88772/AX88179/AX88279 chipsets.
All of them [well supported](https://oracle.github.io/kconfigs/?config=USB_RTL8152&config=USB_RTL8150&config=USB_NET_AX88179_178A&config=USB_NET_AX8817X)
across Linux distros (including Debian), they ship the required drivers as kernel modules.
Why would we need such an adapter we have two built-in Ethernet ports, one may ask.
Well, these Airoha NICs are little bit exotic, at least their kernel modules
does not shipped with any distro, including Debian.

Another unfortunate limitation of R3 and R3 mini that they only have one USB Type-A port.
In practice that means we need a USB HUB if we want to plug in both the flash drive
and the Ethernet adapter.
One alternative is a docking station or similar advanced USB HUB, which
have the Ethernet adapter built-in.
It is less of a problem for R3 since it has SD card slot so we can boot from that.
However in the R3 mini case, the only boot media available for us
in the bootstrap phase is an USB flash drive.

To say some positive about the R3 Mini compared to R3, it has a CH340E
USB serial adapter built into it's USB Type-C port.
So we can power the board from a laptop or PC and have access to the serial console too.
For console access, I use `tio` which listen on the console and automatically
attach if the device started, or detach if the cable unplugged.

```bash
sudo tio /dev/ttyUSB0
```


# Booting the R3 mini

It would be nice to discuss here the ARM boot process in details.
That would be a lengthy topic and more importantly I'm lack of the knowledge to do so.
Therefore in the following I only concentrate on how the R3 mini boots,
even if most part of it is just standard ARM boot, no vendor specific tricks.

## Boot media

The R3 mini can only boot from it's eMMC or SPI NAND flash storage.
The bootrom baked into the SoC reads a register and select the boot media
based on it's value.
By default this value set by a physical switch found on the board.

Both eMMC and NAND flash has a pre-defined partition layout expected by the bootrom.
The NAND flash is a little bit simpler memory type than eMMC, and need a
designated volume management layer for services like track memory wearing, bad blocks,
and volume management.
eMMC can manage these by hardware, so after setting up a GPT layout on it
we are good to go.
This is important when we flashing any boot image: eMMC and NAND layouts
are not compatible. As a result, we cant flash the same binary into both.
Instead, we need separate image for eMMC and for NAND and the steps
of formatting layout expected by the bootrom are different.

## The R3 mini bootchain

The board comes with a custom SinoVOIP (vendor) flavored OpenWRT.
It's a bit out-of-date so it worth to replace it with a recent
mainline OpenWRT image, because it is exists since the board released.
Either way, the default bootchain looks something like this:

```text
OpenWRT and other
  BSP bootchain

┌───────────────┐
│    Bootrom    │  burned into the SoC
└───────┬───────┘
        │
┌───────▼───────┐
│    u-boot     │  NAND or eMMC
└───────┬───────┘
        │
┌───────▼───────┐
│     Linux     │  NAND or eMMC or NVMe
└───────┬───────┘
        │
┌───────▼───────┐
│    rootfs     │  NAND or eMMC or NVMe
└───────────────┘
```

I dont want to spoil it just yet, but if we want to boot stock Debian,
we will have a little bit more complicated bootchain at the end.
But that is mainly because we want to keep the OpenWRT as an alternative
to Debian, kind of a recovery system.
Essentially we can dual-boot OpenWRT and Debian if we want to.

# Installation process

## Flash OpenWRT to eMMC

There is a detailed guide on how to do that.
On the NAND we keep the vendor provided image, which is an old customized OpenWRT.
To flash the eMMC we have to boot from the NAND, so let's flip the
physical switch to the NAND options and power on the device.
We need the USB flash drive (FAT filesystem) with the upstream OpenWRT files.
Those are downloaded from OpenWRT firmware selector page.
Not all files needed from there, only those which mentioned in the commands.

I copy here the commands required for the flashing just for reference.
Boot into the OS found in NAND, then connect to USB flash drive and copy the files to `/tmp`.
As mentioned before, the eMMC layout is fix, therefore every partition must be the same as below.


```bash
dd if=/tmp/openwrt-*-r3-mini-emmc-gpt.bin of=/dev/mmcblk0
# reboot
echo 0 > /sys/block/mmcblk0boot0/force_ro
dd if=/tmp/openwrt-*-bananapi_bpi-r3-mini-emmc-preloader.bin of=/dev/mmcblk0boot0
dd if=/tmp/openwrt-*-bananapi_bpi-r3-mini-emmc-bl31-uboot.fip of=/dev/mmcblk0p3
dd if=/tmp/openwrt-*-bananapi_bpi-r3-mini-initramfs-recovery.itb of=/dev/mmcblk0p4
dd if=/tmp/openwrt-*-bananapi_bpi-r3-mini-squashfs-sysupgrade.itb of=/dev/mmcblk0p5
sync
# reboot again
```

## Get to the bootloader

After booting the device, a boot menu appears for a brief time.
There if `ESC` pressed, we get to the u-boot CLI.
U-boot essentially the industry standard second-stage bootloader
for embedded devices.
It has lots of features, but usually many of them turned off compile time,
so the final bootloader binary will be small, easy to flash on limited space.

By default it would load the Linux kernel image into the memory,
and jump to it to boot the system.
Instead of this, we would like to boot the Debian installer here.
It's a great thing that OpenWRT's u-boot compiled with EFI support.
With that, it's very easy to get to the installer.

## Debian installation

For those who installed Debian as VM or on x86 machine I have good news:
the process basically the same for the R3 mini.
However there will be few tricks for get to the installer,
and don't leave a broken system after the installation.

### Preparations

At the time of writing, Debian Trixie can be installed on R3 mini,
but it is not released yet so we need the RC1 installer DVD.
There are weekly generated DVD images as well, but in my case
that complained about mismatching kernel and module versions.
As the installer media, we will use the USB drive, we need to
format it to __EXT4__ filesystem.
Then simply copy all the files (even the hidden folders) from the mounted
Trixie ISO to the USB.

Now we have the installer files ready, but we will need the device-tree binary,
to boot the system.
Unlike in x86 world where we have more or less the same boot process,
ARM world is bit different.
At x86, hardware components mostly discovered with ACPI probing,
while ARM based platforms prefers hardware descriptions.
These are different for each device, and containing data for Linux
about addressable memory, CPU layout/cores, PCIe, USB hubs, etc.
This descriptor is the standardized and widely-adopted device-tree source (DTS)
text based format, which is compiled to binary (DTB) to be used.
For R3 mini we can download the DTB file separately from here:
https://d-i.debian.org/daily-images/arm64/daily/device-tree/mediatek/mt7986a-bananapi-bpi-r3-mini.dtb
Then copy it to the USB so we can access it when we load the installer.

### Load the installer

Connect the USB to the R3 mini (while we still in the u-boot console)
and load the installer as a EFI payload.
For that we will need the DTB file as well, which will be passed
for GRUB which is the default bootloader of Debian.

```bash
usb start
setenv fdt_addr_r 0x40000000
setenv kernel_addr_r 0x40008000
ext4load usb 0:3 ${fdt_addr_r} mt7986a-bananapi-bpi-r3-mini.dtb
fatload usb 0:2 ${kernel_addr_r} EFI/boot/grubaa64.efi
bootefi ${kernel_addr_r} ${fdt_addr_r}
```

This will boot the familiar GRUB with the installer's menu.
Since we connected with serial console, we only have character interface,
so instead of a screenshot, I simply "copy" the menu below:

```text
/-------------------------------------------------------------\
|                                                             |
|                                                             |
|*Install                                                     |
| Graphical install                                           |
| Advanced options ...                                        |
| Accessible dark contrast installer menu ...                 |
| Install with speech synthesis                               |
|                                                             |
|                                                             |
|                                                             |
|                                                             |
\-------------------------------------------------------------/

Use the ^ and v keys to select which entry is highlighted.
Press enter to boot the selected OS, `e' to edit the commands
```

But here we cannot simply move forward with the installation.
If we try, it will stuck in a few seconds with a kernel panic.
Instead, press 'e' and on the fly edit the "Install" GRUB menu entry.
The default entry is something like this:

```text
setparams 'Install'

    set background_color=black
    linux    /linux --- quiet
    initrd   /initrd.gz
```

We have to ignore the uninitialized clocks.
This is because at the moment the clock drivers for MT7986
are shipped as kernel modules instead built into the kernel.
However we cannot reach the initramfs loading without them.
This is because the kernel expect all the clocks bound to their drivers
before the init part, which can only happens if they built-in.
Chicken or egg problem, but we need to advance further.
There is a good article about the kernel command line parameter we well use
in order to ignore these clocks.
With that in mind, just edit the menu entry like this:

```text
setparams 'Install'

    set background_color=black
    linux    /linux clk_ignore_unused pd_ignore_unused cma=128M console=ttyS0,115200n8
    initrd   /initrd.gz
```

The `console...` parameter is optional, it tells the kernel to output it's
log messages to the serial port.
If not given, we can't see error messages printed by the kernel if something wrong.

If there are no issues, we will reach the Debian Installer after booting
with this entry.

### Installing Debian

The installation is quite straightforward and plenty of guide available online if there are issues.
Few things to look for:

* The DVD installer has almost every driver modules included,
we are fine with offline installation, no network needed
* Even with that, it worth to connect the R3 mini to the network
with the USB Ethernet dongle, so it can download up-to-date packages
* In my case, the built-in WLAN and Ethernet was unavailable. That is because the WLAN driver is included
in the DVD, but the `linux-firmware` package not. Same for the Ethernet, but there not even the driver included.
* I installed the system into a NVMe SSD. This was intentional, to be able to dual-boot with OpenWRT.

### Post-installation

When the installer finished it's job, do not reboot the system!
We have to edit `/etc/default/grub` and `/boot/grub/boot.cfg` as we did
in the installer menu, to ignore the uninitialized clocks.

GRUB_CMDLINE_LINUX_DEFAULT="quiet clk_ignore_unused pd_ignore_unused cma=128M"
