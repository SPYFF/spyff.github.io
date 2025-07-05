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

Another unfortunate limitation of R3 mini that it only has one USB Type-A port.
In practice that means we need a USB HUB if we want to plug in both the flash drive
and the Ethernet adapter.
One alternative is a docking station or similar advanced USB HUB, which
have the Ethernet adapter built-in.

To say some positive about the R3 Mini compared to R3, it has a CH340E
USB serial adapter built into it's USB Type-C port.
So we can power the board from a laptop or PC and have access to the serial console too.

