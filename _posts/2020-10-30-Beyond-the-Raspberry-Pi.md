---
author: matthias_luescher
author_profile: true
description: "Many great products start on the Raspberry Pi - and get pushed out into the public with a lousy, handmade and not sustainable setup. Luckily a proper industrialization of the product is easy to do and will save you a lot of troubles in the long run."
comments: true
title: "Beyond the Raspberry Pi"
---

Imagine you have a great product idea and you start to implement it on the ubiquitous
[Raspberry Pi](https://www.raspberrypi.org/). Since you are already familiar
with the [Debian operating system](https://www.debian.org/) you make quick progress.
The hardware setup is still hand wired and even laying around on your desk - unprotected and without any cover.
This is fine as it is only a prototype. The operating system is hand adjusted according to the needs of the
product. The software starts to look great and - from a feature point of view - you are almost tempted to sell a minimal
viable product. But hold on!

Although you have done everything right so far there is a huge risk that you make the same mistake that a lot of
others have made already.

Take a few days to properly industrialize your product ...

- ... by choosing a rugged hardware platform that supports secure boot,
- ... by properly setting up your operating system in a reproducible manner,
- ... by preparing a partition layout that allows you to upgrade the entire operating system in a robust way and
- ... by properly documenting your software releases.

Of course you want to keep the Debian setup because ...

- ... you got used to it,
- ... you like it,
- ... you learned that there is [long term support](https://wiki.debian.org/LTS) including security updates and
- ... and you just cannot afford to wait any longer to get your product out into the market.

The good news is that there is an easy way forward. Here is my proposal:

Rugged Hardware
---------------

The environment where you install your embedded device is often harsher than your office desk. Therefore, it makes sense
to get a device that can reliably deal with such an environment in 24/7 operation mode. Furthermore you might want to keep the
hardware diversity low and therefore it makes a lot of sense to buy a device that can still be bought in 10 years.

With the given constraints the
[Compulab IOT-GATE-iMX8](https://www.compulab.com/products/iot-gateways/iot-gate-imx8-industrial-arm-iot-gateway/) could be a
perfect match for you:

![Compulab IOT-GATE-iMX8](/assets/images/blog/IOT-GATE-iMX8.png){:class="img-responsive"}

The CPU power is similar to the Raspberry Pi 3 and there are plenty of I/O options. Even cellular 4G connectivity is
possible. It is based on the NXP i.MX platform - which is probably one of the most well documented embedded platforms when it comes 
to secure boot. Also the
[device specific documentation](https://www.compulab.com/products/iot-gateways/iot-gate-imx8-industrial-arm-iot-gateway/#devres)
is very decent and up to date.

Recent Software
---------------

Unfortunately more often than not you get an embedded ARM CPU with a terribly outdated boot loader and Linux kernel setup. This is
not the case here as NXP and Compulab are perfectly committed to deliver their products with recent
[LTS kernels](https://www.nxp.com/design/software/embedded-software/i-mx-software/embedded-linux-for-i-mx-applications-processors:IMXLINUX)
and an up to date U-Boot boot loader.
Naturally the operating system setup is based on the latest Debian release.

Reproducible Operating System Setup
-----------------------------------

Maybe you have started your project with a Raspberry Pi OS Lite image and then manually adjusted it according to your needs.
Of course you could now rip out the SD card and use the `dd` command to reproduce the setup on a bunch of other devices.
However, in the long run this will turn out to be not sustainable.

A single command that produces the entire operating system image in a reproducible manner would be ideal - and this is
exactly what you can [find here - tailored for the Compulab IOT-GATE-iMX8](https://github.com/lueschem/edi-cl/).

When following the [given instructions](https://github.com/lueschem/edi-cl/#basic-usage) you will notice,
that a single command is sufficient to produce a fully working operating system image for the Compulab IOT-GATE-iMX8:

``` bash
sudo edi -v image create iot-gate-imx8-buster-arm64.yml
```

The generated image is even clever enough to boot either from a USB stick (development use case) or from the built in eMMC 
(production use case).

You can fork the [edi-cl repository](https://github.com/lueschem/edi-cl/) and tailor it according to your needs.

Robust Updates
--------------

On Debian it is very tempting to purely rely upon apt to do operating system updates. But imagine if you want to maintain a fleet
of devices from day one by just using apt. Your root file system will get polluted with the leftovers of a long incremental upgrade
history. At some point you might even brick your system because you run into a corner case that you have never seen before.

Leaving behind all the hacks from the early days of the project is exactly what you want to do after some months or years. Luckily
the operating system image you have built above comes with exactly this functionality:

![Robust A/B Update Setup](/assets/images/blog/cl_partition_layout.png){:class="img-responsive"}

It makes use of [Mender](https://mender.io/) for robust A/B based updates. While running the OS from the A partition
you can stream a new OS version onto the B partition. Once this is done the device gets rebooted and will start from the B
partition. If the B partition boots properly, then you are fine, and you just got rid of all the upgrade history. 
If something goes wrong, then the system will reboot again and revert to the A partition.

Configurations that shall be sticky can be backed up to the data partition and get restored from there. Mender supports this
through [state scripts](https://docs.mender.io/artifact-creation/state-scripts).

Digital Twin for Development
----------------------------

Sure - you can do the development of your application on the Raspberry Pi. But after some time, you will get bored as it is still a
lot faster to develop on your AMD Ryzen or Intel Core i7. This is exactly the reason why [edi](https://www.get-edi.io)
comes with the "digital twin" offering:

![Digital Twin](/assets/images/blog/digital_twin.png){:class="img-responsive"}

The digital twin LXD container uses the same Debian release as the target device but inherits the CPU architecture of the host
system (typically amd64). It is even possible to connect real hardware to the container using USB and network device pass-through.
A cross compiler is preinstalled and can be used to build the kernel or applications and libraries.
Once you got used to the setup you might end up doing most of your development work within the digital twin.

Conclusion
----------

The Raspberry Pi heavily contributed to the popularity of Debian within the embedded space. It is a great platform with a fantastic
community and ecosystem. Many great product ideas get started on the Raspberry Pi - and there is nothing wrong here.
However, if the idea gets transformed into a product then some additional factors should be taken into account.
The above blog post outlines an approach that has been successfully implemented and applied to many embedded devices.
It all started on the ubiquitous Raspberry Pi and in my case ended up on the NXP i.MX platform that is well suited for
industrial applications.

