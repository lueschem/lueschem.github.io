---
author: matthias_luescher
author_profile: true
description: "This blog post outlines how a Debian based system can be properly booted using U-Boot."
comments: true
title: "Booting Debian with U-Boot"
---

Several sources
(e.g. [IoT Developer Survey - slide 29](https://www.slideshare.net/kartben/iot-developer-survey-2018), 
[Samsung Artik - see firmware](https://developer.artik.io/documentation/downloads.html) and 
[Azure IoT Edge](https://docs.microsoft.com/en-us/azure/iot-edge/support)) 
indicate that Ubuntu/Debian based systems are a very popular choice for
IoT gateways. On the other hand [U-Boot](https://www.denx.de/wiki/U-Boot)
is the de facto standard for booting embedded devices. This blog post will outline how you can
elegantly combine those two popular choices.

![boot](/assets/images/blog/boot.png){:class="img-responsive"}

Having a proper boot loader setup for
your IoT device is a mandatory prerequisite for reliable updates.

## Requirements

I have been experimenting quite a lot with the Raspberry Pi 
[boot loader](https://github.com/raspberrypi/firmware/tree/master/boot) and its [Debian
integration](https://salsa.debian.org/debian/raspi3-firmware). 
The Raspberry Pi boot loader shines with its simpleness - which is perfect 
for its intended educational purpose.

However having a big IoT fleet in mind, I came to the conclusion that this setup can not
fulfill all my requirements:

- Boot standard Debian kernels without moving around files (such as ram disks, 
device tree binaries, kernel images).
- "Atomically" switch from one kernel version to another one.
- Support a multi boot scenario with a fail safe fallback.
- Be configurable according to the use case.
- Handle everything automatically during unattended updates.

The following explanations will show how I tweaked my 
[Debian based Raspberry Pi image generation](https://github.com/lueschem/edi-pi) to support
the above requirements.

## Building the Image

Given you have installed the tool `edi` according to
[this instructions](https://docs.get-edi.io/en/latest/getting_started.html)
(please take a careful look at the "Setting up ssh Keys" section since you
will need a proper ssh key setup in order to access the Rasperry Pi using ssh) 
and cloned the *edi-pi* project configuration repository from GitHub:

``` bash
git clone https://github.com/lueschem/edi-pi.git
git checkout stretch
cd edi-pi
``` 

and installed a few extra tools:

``` bash
sudo apt install e2fsprogs dosfstools bmap-tools
```

You are now ready to generate a minimal pure Debian stretch arm64
image for the Raspberry Pi 3:

| Important Note |
| --- |
| OS image generation operations require superuser privileges and therefore you can easily break your host operating system. Please ensure that you have a backup copy of your data. |

``` bash
sudo edi -v image create pi3-stretch-arm64.yml
```

The resulting image can be copied to an *unmounted* SD card (here `/dev/mmcblk0`)
using the following command:

| Important Note |
| --- |
| Everything on the SD card will be erased! |

``` bash
sudo bmaptool copy artifacts/pi3-stretch-arm64.img /dev/mmcblk0
```

Once you have booted the Raspberry Pi 3 using this SD card you can
access it using ssh (the access should be granted thanks to your
ssh keys):

``` bash
ssh pi@IP_ADDRESS
```

or via local login using keyboard and monitor (the password for the user
_pi_ is _raspberry_).

In the next section we will take a close look at the boot procedure of the image we have 
just generated.

## Behind the Scenes

When you power on the Raspberry Pi the 
[standard boot loader](https://github.com/raspberrypi/firmware/tree/master/boot) on the 
vfat partition (see `/boot/firmware`) will kick in. However, it will not directly load Linux 
but instead load the [U-Boot boot loader](https://www.denx.de/wiki/U-Boot) also located on the 
vfat partition according to `/boot/firmware/config.txt`:

```
# Switch the CPU from ARMv7 into ARMv8 (aarch64) mode
arm_control=0x200

enable_uart=1

kernel=u-boot.bin
```

As soon as U-Boot has started we find ourselves in an environment that is familiar to most 
embedded Linux developers. 

In the next step U-Boot will look for a file called `boot.scr` 
and boot the system accordingly. The instructions within `boot.scr` could look like this:

```
setenv bootargs console=tty0 console=ttyS1,115200 root=/dev/mmcblk0p2 rw elevator=deadline fsck.repair=yes net.ifnames=0 cma=128M rootwait

ext4load mmc 0:2 ${kernel_addr_r} /boot/vmlinuz-4.18.0-0.bpo.1-arm64
ext4load mmc 0:2 ${fdt_addr_r} /usr/lib/linux-image-4.18.0-0.bpo.1-arm64/${fdtfile}
ext4load mmc 0:2 ${ramdisk_addr_r} /boot/initrd.img-4.18.0-0.bpo.1-arm64

booti ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r}
```

On the first line we configure the kernel's command-line parameters. The second line will load the 
kernel image from the _ext4_ partition. The following two lines will load the matching device 
tree binary and the initial ram disk from the locations where the Debian package has placed them.
Finally the `booti` (or `bootz` for armhf systems) command will boot into the loaded Debian 
system.

To bridge the gap between U-Boot and Debian I have developed a small package called 
[edi-boot-shim](https://github.com/lueschem/edi-boot-shim).
This package contains the small script `edi-boot-shim` (located in `/usr/bin`) that can be told 
to updated the boot instructions within `boot.scr`. By executing the following command, you will
make sure that the given kernel package is getting used for the next boot:

``` bash
sudo edi-boot-shim linux-image-4.18.0-0.bpo.1-arm64
```

In the case of an unattended update you might not want to call this command directly but rather 
rely upon a hook script that gets triggered during kernel updates. The `edi-boot-shim` package 
provides such a hook script (see `/etc/kernel/postinst.d/zz-edi-boot-shim`).

Finally a given use case might need some customized configuration. The behavior of the package 
`edi-boot-shim` can be fine tuned by adapting its configuration file 
`/etc/edi-boot-shim/edi-boot-shim.cfg` or by adjusting the boot command template 
(e.g. `/etc/edi-boot-shim/boot.cmd.rpi.arm64`).

## Conclusion

Within the above setup the small Debian package `edi-boot-shim` bridges the gap between Debian 
and U-Boot. It should be pretty easy to adapt this shim package to any other board where you have 
a U-Boot bootloader and you would like to start Debian.

Now that we have a powerful boot loader setup it is time to figure out how we can provide 
highly reliable over the air updates with minimal down times and fallback scenarios.

As many people have figured out, a `sudo apt update && sudo apt upgrade` might not be reliable 
enough for managing a big IoT fleet.

Integrating one of the following approaches looks like a promising next step:

- [RAUC](https://rauc.io): Safe and secure software updates for embedded Linux
- [SWUpdate](https://github.com/sbabic/swupdate): Software Update for Embedded Systems
- [OSTree](https://ostree.readthedocs.io/en/latest/)


