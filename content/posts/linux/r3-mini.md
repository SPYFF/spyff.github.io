---
categories: ["linux", "router"]
date: 2025-08-01T07:00:00Z
summary: Installing stock Debian into Banana Pi BPI-R3 Mini router board
title: BPI-R3 Mini with stock Debian
slug: "r3-mini"
canonical: "https://fejes.dev/posts/linux/r3-mini/"
draft: false
---

# Intro

Banana Pi BPI-R3 Mini is a palm-sized router/development board.
It's quite powerful, with a Mediatek MT7986 quad-core CPU and 2GB of RAM.
With 2x2.5GbE and 2.4/5GHz 802.11ax Wi-Fi, it's a good candidate for a portable router.
However, in this post, I would like to go beyond simply flashing OpenWRT into it and calling it a day.

My main goal is to understand how it boots, then install stock Debian into it.
By "stock," I mean the installer officially provided by the Debian project,
no modifications or custom-built debootstrap images.
Ideally, I want to keep the OpenWRT system as well (just in case) and
install Debian into the NVMe SSD, booting it as the default system.
Yes, despite the small size, it has two M.2 slots, and one of them is
PCIe while the other is USB, so one can equip the board with an NVMe SSD and a 5G modem.

## Available resources

The great thing about this board is the plenty of available official and community-maintained documentation.
Also, it has official and unofficial BSPs (Board Support Package - 
essentially a customized Linux kernel and rootfs tailored to the board).
If one doesn't want to bother with the tweaking described in the following,
it's very easy to get the board up and running with these.

* Vendor's page with useful links [here](https://docs.banana-pi.org/en/BPI-R3_Mini/BananaPi_BPI-R3_Mini)
* OpenWRT database entry [here](https://openwrt.org/toh/sinovoip/bananapi_bpi_r3_mini)
* BananaPi forum topics for this board [here](https://forum.banana-pi.org/c/banana-router/bpi-r3/64)
* Frank Wunderlich's wiki page with many useful info [here](https://www.fw-web.de/dokuwiki/doku.php?id=en:bpi-r3mini:start)

## About the BPI-R3 Mini

As one might guess from the name “mini,” there is a conventional router form factor version of the board. 
This packs more Ethernet ports, even two 10GbE. What is more interesting, it has an SD card slot, which makes experimenting much more convenient: if something goes wrong with our kernel config or rootfs, simply unplug the SD card, flash the new image into it, and retry.

Unfortunately, the R3 mini does not have an SD card slot. 
It has an eMMC and SPI flash soldered onto the board, and that's all. 
This isn't a huge problem, but we have to be a bit more careful, as if something goes wrong, we can end up with a soft-bricked device that takes some effort to recover (see later).

We will need a USB flash drive and a USB-Ethernet dongle for our experiments. 
These USB-Ethernet dongles most commonly use Realtek 8152/8153/8156 or ASIX AX88772/AX88179/AX88279 chipsets. 
All of them are [well supported](https://oracle.github.io/kconfigs/?config=USB_RTL8152&config=USB_RTL8150&config=USB_NET_AX88179_178A&config=USB_NET_AX8817X) across Linux distros (including Debian), as they ship with the required drivers as kernel modules. 
Why would we need such an adapter when we have two built-in Ethernet ports, one may ask? 
Well, these Airoha NICs are a little bit exotic, at least their kernel modules don’t ship with any distro, including Debian.

Another unfortunate limitation of the R3 and R3 mini is that they only have one USB Type-A port. 
In practice, that means we need a USB hub if we want to plug in both the flash drive and the Ethernet adapter. 
One alternative is a docking station or similar advanced USB hub, which has the Ethernet adapter built-in. 
It’s less of a problem for the R3 since it has an SD card slot, so we can boot from that. 
However, in the R3 mini case, the only boot media available to us in the bootstrap phase is a USB flash drive.

To say something positive about the R3 Mini compared to the R3, it has a CH340E USB serial adapter built into its USB Type-C port. 
So, we can power the board from a laptop or PC and have access to the serial console too. 
For console access, I use `tio`, which listens on the console and automatically attaches if the device starts or detaches if the cable is unplugged.

```bash
sudo tio /dev/ttyUSB0
```

# Booting the R3 mini

It would be nice to discuss the ARM boot process in detail here. 
That would be a lengthy topic, and more importantly, I lack the knowledge to do so. 
Therefore, in the following, I only concentrate on how the R3 mini boots,
even if most of it is just standard ARM boot, with no vendor-specific tricks.

## Boot media

The R3 mini can only boot from its eMMC or SPI NAND flash storage. 
The bootrom baked into the SoC reads a register and selects the boot media based on its value. 
By default, this value is set by a physical switch found on the board.

Both eMMC and NAND flash have a pre-defined partition layout expected by the bootrom. 
The NAND flash is a simpler memory type than eMMC and needs a designated 
volume management layer for services like tracking memory wear, bad blocks, and volume management. 
eMMC can manage these by hardware, so after setting up a GPT layout on it, we are good to go. 
This is important when flashing any boot image: eMMC and NAND layouts are not compatible.
As a result, we can’t flash the same binary into both.
Instead, we need separate images for eMMC and NAND, and the steps of
formatting the layout expected by the bootrom are different.

## The R3 mini bootchain

The board comes with a custom SinoVOIP (vendor) flavored OpenWRT. 
It’s a bit out-of-date, so it’s worth replacing it with a recent mainline OpenWRT image,
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

I don’t want to spoil it just yet, but if we want to boot stock Debian,
we will have a little bit more complicated bootchain at the end. 
But that is mainly because we want to keep the OpenWRT as an alternative to Debian, kind of a recovery system. 
Essentially, we can dual-boot OpenWRT and Debian if we want to.

# Installation process

## Flash OpenWRT to eMMC

There is a detailed guide on how to do that. 
On the NAND, we keep the vendor-provided image, which is an old customized OpenWRT. 
To flash the eMMC, we have to boot from the NAND, so let’s flip the physical switch to the NAND option and power on the device. 
We need the USB flash drive (FAT filesystem) with the upstream OpenWRT files. 
Those are downloaded from the OpenWRT firmware selector page. 
Not all files are needed from there, only those mentioned in the commands.

I copy the commands required for the flashing just for reference. 
Boot into the OS found in NAND, then connect to the USB flash drive and copy the files to `/tmp`. 
As mentioned before, the eMMC layout is fixed, therefore every partition must be the same as below.

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
If `ESC` is pressed, we get to the u-boot CLI. 
U-boot is essentially the industry-standard second-stage bootloader for embedded devices. 
It has lots of features, but usually many of them are turned off at compile time,
so the final bootloader binary is small and easy to flash on limited space.

By default, it would load the Linux kernel image into memory and jump to it to boot the system. 
Instead, we want to boot the Debian installer here. 
It’s a great thing that OpenWRT’s u-boot is compiled with EFI support. 
With that, it’s very easy to get to the installer.

## Debian installation

For those who installed Debian as a VM or on an x86 machine, I have good news: 
the process is basically the same for the R3 mini. 
However, there will be a few tricks to get to the installer and to avoid leaving a broken system after the installation.

### Preparations

At the time of writing, Debian Trixie can be installed on the R3 mini, but it is not released yet, so we need the RC1 installer DVD. 
There are weekly generated DVD images as well, but in my case, that resulted in
complaints about mismatching kernel and module versions. 
As the installer media, we will use the USB drive, which we need to format to an __EXT4__ filesystem. 
Then, simply copy all the files (even the hidden folders) from the mounted Trixie ISO to the USB.

Now we have the installer files ready, but we will need the device-tree binary to boot the system. 
Unlike in the x86 world, where we have more or less the same boot process, the ARM world is a bit different. 
At x86, hardware components are mostly discovered with ACPI probing,
while ARM-based platforms prefer hardware descriptions. 
These are different for each device and contain data for Linux about addressable memory, CPU layout/cores, PCIe, USB hubs, etc. 
This descriptor is the standardized and widely-adopted device-tree source (DTS) text-based format,
which is compiled to binary (DTB) to be used. 
For the R3 mini, we can download the DTB file separately from 
[here](https://d-i.debian.org/daily-images/arm64/daily/device-tree/mediatek/mt7986a-bananapi-bpi-r3-mini.dtb)
Then, copy it to the USB so we can access it when we load the installer.

### Load the installer

Connect the USB to the R3 mini (while we’re still in the u-boot console) and load the installer as an EFI payload. 
For that, we will need the DTB file as well, which will be passed to GRUB, which is the default bootloader of Debian.

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

But here we cannot simply proceed with the installation. 
If we try, it will get stuck in a few seconds with a kernel panic. 
Instead, press 'e' and edit the "Install" GRUB menu entry on the fly. 
The default entry is something like this:

```text
setparams 'Install'

    set background_color=black
    linux    /linux --- quiet
    initrd   /initrd.gz
```

We have to ignore the uninitialized clocks.
This is because, at the moment, the clock drivers for MT7986 are shipped as
kernel modules instead of being built into the kernel.
However, we cannot reach the initramfs loading without them.
This is because the kernel expects all the clocks to be bound to their drivers
before the init phase, which can only happen if they are built-in.
It’s a classic chicken or egg problem, but we need to advance further.
[There](https://blog.dowhile0.org/2024/06/02/some-useful-linux-kernel-cmdline-debug-parameters-to-troubleshoot-driver-issues/)
is a good article about the kernel command line parameter we will use in order to ignore these clocks.
With that in mind, just edit the menu entry like this:

```text
setparams 'Install'

    set background_color=black
    linux    /linux clk_ignore_unused pd_ignore_unused cma=128M console=ttyS0,115200n8
    initrd   /initrd.gz
```

The `console...` parameter is optional; it tells the kernel to output its
log messages to the serial port.
If it's not given, we can't see error messages printed by the kernel if something goes wrong.

If there are no issues, we will reach the Debian Installer after booting
with this entry.

### Installing Debian

The installation is quite straightforward, and plenty of guides are available online if there are issues.
A few things to look for:

* The DVD installer has almost every driver module included,
so we are fine with offline installation; no network is needed.
* Even with that, it's worth connecting the R3 mini to the network
with the USB Ethernet dongle, so it can download up-to-date packages.
* In my case, the built-in WLAN and Ethernet were unavailable. This is because the WLAN driver is included
on the DVD, but the `linux-firmware` package is not. The same goes for Ethernet, but there’s not even the driver included.
* I installed the system onto an NVMe SSD. This was intentional, to be able to dual-boot with OpenWRT.

### Post-installation

When the installer finishes its job, do not reboot the system!
We have to edit `/etc/default/grub` and `/boot/grub/boot.cfg` as we did
in the installer menu, to ignore the uninitialized clocks.

For that, press `Ctrl+A` and `2` to switch from the installer’s screen to the shell screen.
Here, edit `/boot/grub/boot.cfg` with `nano` (since `vi` or `vim` are not included in the installer).
Search for `menuentry 'Debian GNU/Linux'...`, which is usually the first entry,
above `menuentry 'Advanced options for Debian GNU/Linux'...`.
Edit or copy the entry entirely and create a new menu entry something like this
(some text omitted where `...`):

```
menuentry 'R3 mini Debian GNU/Linux' --class debian ... {
	insmod part_gpt
	insmod ext2
...
	linux	/boot/vmlinuz root=UUID=... console=ttyS0,115200n8 clk_ignore_unused pd_ignore_unused cma=128M
	initrd	/boot/initrd.img
  devicetree /boot/mt7986a-bananapi-bpi-r3-mini.dtb
}
```

This entry right now is only temporary, only required for the first boot.
The important part here, aside from ignoring unused clock drivers, is the
`devicetree` line, which tells GRUB the path of the DTB that should be passed to the kernel.
But it’s not there, so first, mount the root partition from the NVMe SSD,
and copy the DTB from the USB to the `/boot` folder.

To make these changes permanent, we need to add the following line
to the `/etc/default/grub`:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet clk_ignore_unused pd_ignore_unused cma=128M"
```

This does not have an immediate effect; instead, it modifies the GRUB configuration generation
to append these booting parameters to the kernel arguments.
Therefore, after we boot into the Debian system, and run the `update-grub` command or
update the kernel, these arguments will appear in the newly generated `grub.cfg`.
But we’re not there yet…

## Fixing the boot

At the moment, we have Debian installed on the NVMe SSD, but it’s impossible to boot it.
This is because OpenWRT nor upstream u-boot (at least right now) supports PCIe of the R3 mini.
There’s a [patch](https://patchwork.ozlabs.org/project/uboot/patch/20240412141051.23943-1-linux@fw-web.de/) submitted for that,
but that’s only an RFC, not intended for merge.

Frank Wunderlich maintains a fork of u-boot with PCIe enabled.
I [forked](https://github.com/SPYFF/u-boot/releases) this, since I wanted to enable some other features like EFI boot and EXT4 filesystem.
With this u-boot configuration, we can boot from the NVMe.
It would be nice if we could do this with the stock OpenWRT u-boot,
one way to do that would be to keep `/boot` on the eMMC and the rest of the rootfs on the NVMe.
But let’s keep this for future work, maybe the subject of another post.

### Modified bootchain

```text
Modified bootchain
    for Debian

┌───────────────┐ 
│    Bootrom    │  burned into the SoC
└───────┬───────┘
        │
┌───────▼───────┐
│ u-boot, OWRT  │  eMMC
│     stock     │
└───────┬───────┘
        │
┌───────▼───────┐
│ u-boot, fork  │  USB flashdrive
└───────┬───────┘
        │
┌───────▼───────┐
│     GRUB      │  NVMe
└───────┬───────┘
        │
┌───────▼───────┐
│     Linux     │  NVMe
└───────────────┘
```

This bootchain is complicated, but it doesn't affect the original OpenWRT
installation and allows dual-booting Debian as well.
First, the stock OpenWRT u-boot booted, which only chainloads the forked u-boot with PCIe, EXT4, and EFI boot support.
For convenience, this u-boot binary is placed on a USB flash drive, so it can be replaced easily if needed.
Next, with EFI boot, this boots the GRUB installed by Debian.
Previously, we modified `grub.cfg` so that it can now load the kernel and boot the system.
To achieve this, we need the following new, persistent u-boot environment configuration.

```bash
setenv bootmenu_11 'Chainload from u-boot from USB.=run boot_usb_chain ; run bootmenu_confirm_return'
setenv bootmenu_default '11'
setenv bootmenu_delay '5'
setenv boot_usb_chain 'led $bootled_pwr on ; usb start ; fatload usb 0:1 $loadaddr uboot.bin ; go $loadaddr ; led $bootled_pwr off'
env save
```

The first line create a new menu entry in OpenWRT default bootmenu,
the second line set this entry to the default.
The forth line initialize the u-boot USB subsystem, load the our customized fork u-boot's
binary `uboot.bin` to the memory and jumps into it.
Lastly, `env save` makes these changes persistent.

In this u-boot, we also need some modifications:

```bash
setenv bootmenu_4 '5. Boot Debian from NVMe (EFI).=run bootnvmeefi_deb'
setenv bootmenu_default '4'
setenv bootnvmeefi_deb 'usb start ; pci enum ; nvme scan ; setenv kernel_addr_r 0x40008000 ; setenv fdt_addr_r 0x40000000 ; fatload usb 0:1 ${fdt_addr_r} mt7986a-bananapi-bpi-r3-mini.dtb ; fatload nvme 0:2 ${kernel_addr_r} EFI/debian/shimaa64.efi ; bootefi ${kernel_addr_r} ${fdt_addr_r}'
env save
```

The most important aspect is the lengthy line.
It loads the devicetree from USB, the GRUB EFI payload from the NVMe, and finally boots it.
From that point, we'll have a normal GRUB to Linux bootchain, which is what we'd expect in Debian/Ubuntu.

In the GRUB menu, we have to choose the `R3 mini Debian GNU/Linux` entry, which was created in the post-installation step of the guide. If everything is OK, the board should boot Debian successfully.

### On the First Boot of Debian

The bootmenu entry used to load Linux was only a temporary entry in `grub.cfg`.
This is because `grub.cfg` is automatically generated on each kernel update,
based on the rules defined in the defaults file: `/etc/default/grub`.
To make this entry resistant to kernel updates, we have to extend the kernel parameters as shown below:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet clk_ignore_unused pd_ignore_unused cma=128M"
GRUB_CMDLINE_LINUX="quiet clk_ignore_unused pd_ignore_unused cma=128M"
```

If only `GRUB_CMDLINE_LINUX_DEFAULT` is modified, only the default menuentry will be extended with these parameters.
With that, it's safe to execute `sudo update-grub` to manually trigger a `grub.cfg` generation.
Before rebooting, verify that the new parameters are included in all menuentries:

```bash
cat /boot/grub/grub.cfg | grep linux | grep clk
```

If the output is not empty, it's safe to reboot the board and, in the future,
simply choose the default menuentry.
Kernel updates shouldn't disrupt the boot process either.
