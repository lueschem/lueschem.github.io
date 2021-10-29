---
author: matthias_luescher
author_profile: true
description: "The edi-pi project configuration got upgraded from Debian buster to Debian bullseye. There are a few noteworthy changes!"
comments: true
title: "Debian Bullseye Upgrade for the Raspberry Pi"
---

About every second year there is a new Debian release. This time it is
[Debian bullseye](https://www.debian.org/releases/bullseye/index.en.html) and therefore an update of the
[edi-pi](https://github.com/lueschem/edi-pi) project configuration was imminent.

However, there was an issue that bugged me more than the release upgrade:
[Fix the dtb handling during A/B update](https://github.com/lueschem/edi-pi/issues/23).

Time to solve it:

New Bootloader Integration for the Pi 4
---------------------------------------

The [edi-pi](https://github.com/lueschem/edi-pi) project configuration comes with a [Mender](https://mender.io/)
enabled A/B partition setup that enables the devices to get updated over the air in a robust manner. However, for the
Raspberry Pi 4 there was an unsolved [issue](https://github.com/lueschem/edi-pi/issues/23)
that made it impossible to swap the device tree binaries and the kernel at
the very same time as kind of an "atomic operation". Many industry and longevity focused projects struggled with similar
setup issues and came up with various workarounds. Luckily the Raspberry Pi foundation has listened to its customers and
implemented the yet pretty undocumented "tryboot" feature. The good news is that this "tryboot" feature can now take
over the fail-safe handling that was previously done by U-Boot
([bootcount implementation](https://www.denx.de/wiki/DULG/UBootBootCountLimit)).
This led to an implementation that comes with a small pi-uboot script that makes Mender client believe it is still
dealing with U-Boot:

![Bootloader Integration](/assets/images/blog/pi-bootloader-integration.png){:class="img-responsive"}

The boot partition looks as follows:

``` bash
pi@raspberry:~$ tree /boot/firmware/
/boot/firmware/
|-- config.txt
|-- p3
|   |-- bcm2711-rpi-4-b.dtb
|   |-- cmdline.txt
|   |-- overlays
|   |   |-- gpio-ir-tx.dtbo
|   |   |-- gpio-ir.dtbo
|   |   `-- vc4-fkms-v3d.dtbo
|   `-- vmlinuz
|-- p3fixup4.dat
|-- p3start4.elf
|-- p4
|   |-- bcm2711-rpi-4-b.dtb
|   |-- cmdline.txt
|   |-- overlays
|   |   |-- gpio-ir-tx.dtbo
|   |   |-- gpio-ir.dtbo
|   |   `-- vc4-fkms-v3d.dtbo
|   `-- vmlinuz
|-- p4fixup4.dat
`-- p4start4.elf
```

The config.txt (or eventually the tryboot.txt) has the following content:

``` bash
pi@raspberry:~$ cat /boot/firmware/config.txt 
# needs to be 8.3 format and in top level folder
start_file=p4start4.elf
fixup_file=p4fixup4.dat

# Switch the CPU from ARMv7 into ARMv8 (aarch64) mode
arm_64bit=1

enable_uart=1

kernel=vmlinuz
os_prefix=p4/
```

The above setup shows, that not only the kernel (vmlinuz), the device tree (bcm2711-rpi-4-b.dtb + overlays) but also
important bootloader components (pXstart4.elf, pXfixup4.dat) get upgraded in a robust manner. Only the bootloader EEPROM
remains unchanged. The Linux kernel is installed through a standard Debian package and
[edi-boot-shim](https://github.com/lueschem/edi-boot-shim) makes sure that the kernel and the device tree binaries end
up on the right partition. The U-Boot binary is no longer needed. All the fine-tuning of config.txt and cmdline.txt
can now be modified during an OTA update without the risk of bricking the device.

NetworkManager Replaces ifupdown
--------------------------------

After some evaluation I decided to replace `ifupdown` with NetworkManager:

``` bash
pi@raspberry:~$ nmcli con show
NAME                UUID                                  TYPE      DEVICE 
my-wifi             398c8116-b0ea-4303-993d-5a4f7544e3d2  wifi      wlan0  
Wired connection 1  4cf0b75b-aa66-329d-9445-04eefb76ce24  ethernet  --
```

This is especially beneficial for setups where also ModemManager is involved (e.g. LTE connection). Furthermore,
NetworkManager is faster on reacting to network topology changes (e.g. plugging and unplugging network cables) and it
also integrates with firewalld.

The network settings now get persisted during a full OS update. This guarantees that a Mender update is successful even
if the device is connected to a protected WiFi network.

GUI Passthrough
---------------

The "digital twin" container remains a very important feature of the setup. It is now even by default possible to run
a GUI application within the (development) container:

![GUI application](/assets/images/blog/gui-passthrough.png){:class="img-responsive"}

Further Changes
---------------

There were some additional changes that made it into this updated setup:

- The ssh host keys get persisted during the full OS update.
- A [bug](https://github.com/lueschem/edi-pi/issues/24) that lead to a non-unique machine-id got fixed.
- Various things got aligned with the [edi-var](https://github.com/lueschem/edi-var) and
  [edi-cl](https://github.com/lueschem/edi-cl) project configurations.

Conclusion
----------

Especially for the Raspberry Pi 4 the changes are a breakthrough as now a robust OTA update is possible.
The implementation provided here might also be beneficial for various Yocto projects that are dealing with the
Raspberry Pi.

As usual, also the new Debian release delivers a lot of updates and new features. E.g.
[Podman](https://podman.io/) became part of the Debian package repository. The daemonless container engine approach
might be very interesting for embedded projects!