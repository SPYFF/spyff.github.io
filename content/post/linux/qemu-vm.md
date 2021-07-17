---
categories: ["linux"]
tags: ["linux", "vm", "qemu"]
date: 2018-07-20T15:00:00Z
summary: Few notes and command dump about creating QEMU virtual machine for testing various kernel features.
title: QEMU virtual machine for kernel testing
slug: "linux"
---

# Abstract
Testing kernel on bare-metal machine is time-consuming. Boot failures, screwing up installed distro, slow restart cycles, etc. There is a very convenient method for booting up a freshly compiled kernel with QEMU. You can compile the kernel on your host machine, then simply slide-load that to a virtual machine with your favorite GNU/Linux distribution. In the following, I will show some commands, how to make an easy to use virtual test environment with QEMU and Ubuntu Server cloud image.

# Preparations on the host machine

My host machine running a Ubuntu 18.04 LTS GNU/Linux distribution with a `hwe-edge` kernel which is version 5.0 currently. For kernel compilation, we should have to install some additional packages:

```
sudo apt install git fakeroot build-essential ncurses-dev \
     xz-utils libssl-dev bc bison flex libelf-dev
```

Also, for booting the virtual machine later, we should install the QEMU and some tools for later image preparations:

```
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system \
     bridge-utils virt-manager cloud-utils
```

# Kernel compilation

There are lots of ways to get the kernel source code, for example, clone a repository of it:

```
git clone --depth 1 https://kernel.googlesource.com/pub/scm/linux/kernel/git/davem/net-next.git
```

I use the **net-next** repository right now, but the process is similar to the stable mainline kernel too.

## Kernel configuration

The whole kernel compilation process depends on the `.config` file, which is located at the kernel source root directory (`net-next` in our case). However, this file did not exist by default, we have to make one. Because we compile it to QEMU-KVM, we don't need many device drivers and real hardware related options. The kernel contains a good default configuration for QEMU-KVM, so we can use that:

```
make x86_64_defconfig
make kvmconfig
make -j `nproc --all`
```

## Creating and applying config fragments

This is okay for test if the kernel compiles and boot successfully, but what if we would like to test the full BPF featureset of the kernel. We have `menuconfig` of course, but searching all the BPF settings one-by-one is time-consuming and brings the possibility of skipping some accidentally. To overcome the problem, we will use **config fragments**. One way to create them is to cut out the required settings from `allyesconfig`, in our case the enabled BPF settings.

```
mv .config kvmconf_backup.config
make allyesconfig
grep BPF .config > bpf.config
mv kvmconf_backup.config .config
./scripts/kconfig/merge_config.sh -m .config bpf.config
```

In line **1** we make a backup from our original x86_64 kvmconfig, because we use that later. Then we make the `allyesconfig` in line **2**. This is a special built in configuration, which enable every kernel feature, including modules and drivers. That one takes forever to compile, so we just cut the BPF part (BPF config fragment) and of that in the line **3**. In line **4** we restore the KVM config backup and apply the BPF fragment for that (line **5**). The output of the method is a lightweight config for virtual machine usage, but with all the BPF features enabled.

Now the kernel is ready for compilation, use the `make -j9` command for run on 9 threads.

# Booting the virtual machine

I use Ubuntu cloud image as the basis of my virtual machine. Most of the core packages still included in this one, but also designed to run in a virtual environment, so the desktop manager and many other bloats not included.

```
wget https://cloud-images.ubuntu.com/eoan/current/eoan-server-cloudimg-amd64.img
``` 

Good practice to keep this image untouched and make an overlay image on top of that for later usage. Writing files in the overlay image is persistent but it does not affect the base image. So if we mess up something very badly, we still have the original image and use that without further problems. We can also make overlay image over overlay images, so if we installed and configured the guest system, we can make an overlay image on top of that and if some bad thing happens, we can revert to the last stable version. Let's create the overlay image:

```
qemu-img create -f qcow2 -b eoan-server-cloudimg-amd64.img ubuntu.img
```

This Ubuntu image is untouched, we have to put some configuration into it. `cloud-init` is a method for that. First, we have to create a cloud config file (with very simple yaml syntax) without identity details, hostname, ssh public key, etc. Save this file as `init.yaml` for example:

```
#cloud-config
hostname: ubu 
users:
  - name: myusername
    ssh-authorized-keys:
      - ssh-rsa AAAAB3Nz[...]34twdf/ myusername@pc 
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash
```

The `#cloud-config` comment on the top is mandatory for cloud-init. Now let's build a cloud init image from that file:

```
cloud-localds init.img init.yaml
```

This `init.img` also mounted to the virtual machine, and the preinstalled `cloud-init` tool in the Ubuntu image will take a look into it and configure the required parameters according to that.
Now we should start the virtual machine using our own kernel:

```
sudo qemu-system-x86_64 \
-machine accel=kvm,type=q35 \
-kernel net-next/arch/x86/boot/bzImage \
-append "root=/dev/sda1 single console=ttyS0 systemd.unit=graphical.target" \
-hda ubuntu.img \
-hdb init.img \
-m 2048 \
--nographic \
-netdev user,id=net0,hostfwd=tcp::2222-:22 -device virtio-net-pci,netdev=net0 \
#-nic user,hostfwd=tcp::2222-:22 \
-fsdev local,id=fs1,path=/home/spyff/folder_to_share,security_model=none \
-device virtio-9p-pci,fsdev=fs1,mount_tag=shared_folder
```

* line 1: staring the QEMU with our host architecture
* line 2: sideload the kernel we compiled before for the virtual machine, which will use that instead of his default kernel
* line 3: the overlay image of our untouched Ubuntu cloud image as the rootfs
* line 4: the initialization image for the cloud-init
* line 5: two gigabytes RAM for the VM
* line 6-7: speed up and emulation optimizations
* line 8: forward the virtual machine's SSH port to the host machine's 2222 TCP port
* line 9-10: not necessary, a shared folder between the guest and the host

# Configuration of the guest

To access the guest over SSH we should use the 2222 port of the host:

```
ssh 0 -p2222
```

Now for folder sharing, we have to put the following line into the end of the `/etc/fstab`. In this example, it will mount the `/home/spyff/folder_to_share` from the host into the guest's `/root/shared` folder:

```
shared_folder /root/shared 9p trans=virtio 0 0
```

## Extending the disk space of the guest

By default, the cloud image configured for 2 gigabytes of additional space. This is a virtual limit, so we can extend it for our needs. The following command extend the image (the hard disk in the virtual machine) with 10 gigabytes.

```
qemu-img resize ubuntu.img +10G
```

The `cloud-init` will automatically grow the rootfs to the end of the additional space in next boot. However, we can do that manually if our cloud-init is too old for that:

```
sudo apt install cloud-guest-utils
growpart /dev/sda 1
```

or if that fails, we still able to do that with `parted`

```
parted

(parted) print                                                            
Model: ATA QEMU HARDDISK (scsi)
Disk /dev/sda: 13.1GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
14      1049kB  5243kB  4194kB                     bios_grub
15      5243kB  116MB   111MB   fat32              boot, esp
 1      116MB   13.0GB  12.9GB  ext4

(parted) resizepart 1
End?  [13.0GB]? 13.0GB
```

# Miscellaneous

## Installing kernel headers

The official and most reliable documentation available here: [https://www.kernel.org/doc/Documentation/kbuild/headers_install.txt](https://www.kernel.org/doc/Documentation/kbuild/headers_install.txt)
In our case this looks like the following:

```
make headers_install INSTALL_HDR_PATH=/usr
```

## Installing linux tools

For installing `perf` and `bpftool` (including `libbpf`) do the following:

```
make -C tools/ perf_install prefix=/usr/
make -C tools/ bpf_install
ldconfig
```

## Kernel debugging with GDB (with Ubuntu 19.10 cloud-image)

Start the machine with the command below, where the `-s` open up the QEMU's GDB stub socket at the TCP 1234 port:

```
sudo qemu-system-x86_64 \
-machine accel=kvm,type=q35 \
-hda ubuntu.img \
-hdb init.img \
-m 2048 \
--nographic \
-nic user,hostfwd=tcp::2222-:22 \
-s 
```

To begin with, we have to get the debug symbols for the kernel. This package not available from the default repositories, we need the debug repositories for that:

```
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list

sudo apt install ubuntu-dbgsym-keyring
sudo apt update
```

The debug repository contains the required debug packages, select the matching one for our current kernel

```
sudo apt install linux-image-$(uname -r)-dbgsym
```

Then copy the kernel file to the host machine in order to make it available for GDB to search symbols in it. The required file is `/usr/lib/debug/boot/vmlinux-5.3.0-18-generic` in this case.

By default, kernel address space layout randmization (KASLR and KAISER) is enabled. This feautire prevent GDB to match the addresses in the loaded kernel with the ones defined in the debug file. Edit the `/etc/default/grub.d/50-cloudimg-settings.cfg` file and append `nokaiser nokaslr` to the kernel command line arguments like below:

```
GRUB_CMDLINE_LINUX_DEFAULT="console=tty1 console=ttyS0 nokaiser nokaslr"
```

Then update the GRUB and reboot the guest machine:

```
sudo update-grub
reboot
```

Now on the host machine, start GDB and attach to the remote target

```
gdb ./vmlinux-5.3.0-18-generic
target remote :1234
```

Now the symbols and breakpoints should work but we still unable to see the source lines. For that, download the source files for our kernel `sudo apt install linux-source` then copy the downloaded archive into the host machine `/usr/src/linux-source-5.3.0/linux-source-5.3.0.tar.bz2`. Now we have to tell where the GDB can find the source files. Untar the sources to a directory and set it in the GDB: the original path told by the GDB.

```
set substitute-path /build/linux-IewOGS/linux-5.3.0/ /home/user/linux-source-5.3.0
```
For kernel module debugging, I found this excellent blogpost: [https://medium.com/@navaneethrvce/debugging-your-linux-kernel-module](https://medium.com/@navaneethrvce/debugging-your-linux-kernel-module-21bf8a8728ba)
