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
essentially customized Linux kernel and rootfs tailored to the board)


