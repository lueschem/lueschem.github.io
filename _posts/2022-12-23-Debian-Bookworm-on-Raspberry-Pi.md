---
author: matthias_luescher
author_profile: true
description: "Debian Bookworm will get released somewhere in summer 2023. Here we run its current development state on the Raspberry Pi."
comments: true
title: "Preview: Running Debian Bookworm on the Raspberry Pi"
---

In this blog post I explore [Debian testing](https://wiki.debian.org/DebianTesting) - the current development
state of the next stable Debian distribution. The next stable release will be called
[bookworm](https://wiki.debian.org/DebianBookworm), and it will most likely get released somewhere in Q3 2023:

![Debian Releases](/assets/images/blog/debian_lts_v2.png){:class="img-responsive"}

The experiment is based on [edi-pi](https://github.com/lueschem/edi-pi) which is an edi project configuration that is
capable of building Debian images for the Raspberry Pi 2, 3 and 4. [edi-pi](https://github.com/lueschem/edi-pi) is
already around since 2017 and back then I started with Debian stretch. Later it has been migrated to buster and then to
bullseye. When upgrading from one Debian release to the next one in the past I usually wanted to improve some parts of
the setup in order to make a subsequent upgrade easier. However, due to time pressure, I ended up doing only minimal
changes to the overall setup. This time, things have fundamentally changed. I invested a lot of time into automation
based on GitHub actions:

- All edi related Debian packages get built automatically occasionally using a
[matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) in order to target multiple Debian
releases. The packages get automatically dispatched to [packagecloud](https://packagecloud.io/get-edi/debian).
- [OS images](/Building-and-Testing-OS-Images-with-GitHub-Actions/) are built, dispatched and tested by another
automatic build job.
- Based on a [GitOps approach](/Managing-an-IoT-Fleet-with-GitOps/), I can move my test fleet back and forth
between releases thanks to the A/B setup enabled by [Mender](https://www.mender.io).

This high level of automation gives me a lot of freedom to try things out and test changes quickly on all potentially
affected devices. In fact, I am now able to do continuous integration of my images based on Debian testing.

Implementation
--------------

To get things going I created a [bookworm (now master) branch on edi-pi](https://github.com/lueschem/edi-pi). I
refactored  the setup quite a bit in order to simplify future upgrades. For the Raspberry Pi 4 I am currently using a
kernel that is based on the [source code provided by the Raspberry Pi foundation](https://github.com/raspberrypi/linux).
A [build job](https://github.com/lueschem/edi-ci-public/actions/workflows/kernel-build-rpi4.yml) does build and upload
the package for the appropriate Debian release. [edi-boot-shim](https://github.com/lueschem/edi-boot-shim) - the small
package that does the glue between the bootloader and the kernel - also
[got extended](https://github.com/lueschem/edi-boot-shim/commit/f03b735c26be40677249c6d4fc072674dba65fd5) in order to
provide a package for Debian bookworm. Some third party packages are not yet available for Debian bookworm, and
therefore I decided to add another build job that does
[Mender related packages](https://github.com/lueschem/edi-ci-public/actions/workflows/mender-package-build.yml).

Findings
--------

The differences between Debian bullseye and bookworm are not too fundamental. After only a short time the Raspberry
Pi 2, 3 and 4 devices were booting Debian bookworm successfully. There was even a very positive surprise: The GitOps
enabled Mender full OS update artifact got reduced from 227MB (bullseye) to 170MB (bookworm). This 25% reduction was
made possible thanks to the splitting of the Debian Ansible package (now I am using
[ansible-core](https://packages.debian.org/bookworm/ansible-core)) and because of the refactoring of
[python3-pycryptodome](https://packages.debian.org/bookworm/python3-pycryptodome). Many thanks to the Debian
and Ansible developers that made this possible!

Without any problem the [kiosk terminal setup](/Surprisingly-Easy-IoT-Device-Management/) got applied on top of
Debian bookworm:

![bullseye vs bookworm](/assets/images/blog/bullseye_bookworm.jpg){:class="img-responsive"}

The left screen is still operated by a device that is based on Debian bullseye while the right screen is powered by a
device that is running Debian bookworm.

Conclusion
----------

The Debian community does a fantastic job in juggling with close to 60k packages! Debian testing is pretty stable -
actually stable enough that Google decided to operate
[some desktop machines](https://cloud.google.com/blog/topics/developers-practitioners/how-google-got-to-rolling-linux-releases-for-desktops)
based on this rolling release. In their post they also made the following statement:
"Our journey has ultimately reinforced our belief that incremental changes are better manageable than big bang
releases." I couldn't agree more!
The automation I did for the edi project enabled me to follow the fast pace of a rolling release. If I started a new
project right now, I would even consider to kick it off using Debian testing. At a later stage a switch to a stable
release is straightforward and Debian comes with the luxury of [long term support](https://www.freexian.com/lts/debian/)
or even extended long term support.
