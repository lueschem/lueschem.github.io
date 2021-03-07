---
author: matthias_luescher
author_profile: true
description: "Over the past few years Debian has become a truly viable solution for embedded systems!"
comments: true
title: "Debian for Embedded Systems - Seriously?"
---

On a regular basis I come across "white papers" and presentations that compare the usage of Debian to solution XY for
embedded use cases. There I read claims such as "Debian is only suitable for a quick prototype", "Debian systems are
hard to reproduce", "you can't conveniently cross compile for Debian systems", "long term support is a nightmare"
and "you can't cross bootstrap images for a foreign platform".

Let's debunk those myths for good by giving plenty of evidence that the things have dramatically changed over the last
few years.

For a proper technical analysis we tear apart the components of a typical embedded system:

- **Hardware:** Decent hardware with good mainline and vendor support is key for a successful embedded system.
- **Bootloader:** The first software that runs on the embedded system is the bootloader.
- **Kernel:** The bootloader will load the kernel and hand over the control to it during the startup.
- **Root file system:** The kernel will mount the root file system that contains the applications you want to run on
the embedded system.
- **Tools:** Typically embedded systems are tailored for their exact use case and therefore you need powerful tools
for the tailoring and the application development.
- **Support:** Nowadays embedded systems are complex, and you will not be able to take over the maintenance burden for
the entire system. Therefore, you need an ecosystem that helps you to support your system.
  
Let's go into more details:

Hardware
--------

![Hardware running Debian](/assets/images/blog/hardware.png){:class="img-responsive"}

The embedded world got literally flooded by boards that are capable of running Debian and even come with proper Debian
support. The most prominent representative is the [Raspberry Pi](https://www.raspberrypi.org/) (see also
[pi-gen tool](https://github.com/RPi-Distro/pi-gen) and [rpi23-gen-image tool](https://github.com/drtyhlpr/rpi23-gen-image))
seconded by devices supported by [Armbian](https://www.armbian.com/)
(see also [Armbian github repository](https://github.com/armbian)). Of course, you won't be able to flood the market with a
clunky setup and therefore the above solutions are easy to use and perfectly suitable not only for students that want to learn
more about embedded systems. In many regards established board vendors can learn a lot from the above communities:
ease of use, longevity, documentation, upstreaming, ...

Ok, now you could say that the above boards are for tinkerer. Depending on your use case this might be true and
therefore I will provide a list of industrial grade devices and boards/SOMs running Debian:

- **Moxa IIoT gateways:**
  The [UC-8200 Series](https://www.moxa.com/en/products/industrial-computing/arm-based-computers/uc-8200-series)
  are running Debian based
  [Moxa Industrial Linux](https://www.moxa.com/en/spotlight/industrial-computing/arm-linux-iiot-edge-gateway-portal/linux).
- **Compulab IIoT gateways:** Compulab sells
  [IIoT gateways](https://www.compulab.com/products/iot-gateways/iot-gate-imx8-industrial-arm-iot-gateway/) with Debian
  preloaded.   
- **Variscite SOMs:** Variscite sells
  [SOMs](https://www.variscite.com/product/system-on-module-som/cortex-a53-krait/var-som-mx8m-nano-nxp-i-mx-8m-nano/)
  with comprehensive
  [Debian support](https://github.com/varigit/debian-var).
- **Toradex SOMs:** The Toradex board support packages are built using Yocto, but they decided to support Debian 
  containers through their
  [Torizon platform](https://developer.toradex.com/knowledge-base/torizoncore-overview#TorizonCore_Debian_Containers).
- **Siemens SIMATIC IOT2050:**
  Also the [SIMATIC IOT2050](https://new.siemens.com/global/en/products/automation/pc-based/iot-gateways/simatic-iot2050.html)
  comes with [Debian preloaded](https://github.com/siemens/meta-iot2050).
  
The above vendors are all renowned, and they share the strategy to deliver proper Debian support with their hardware.
I am sure there are plenty of additional good examples.

Verdict: A lot of great, industrial embedded hardware comes with out of the box Debian support.

Bootloader and Kernel
---------------------

No matter which flavor of Linux you bring to your embedded board the situation for the bootloader, and the kernel is the
same: You would like to use the upstream source code ([U-Boot](https://github.com/u-boot/u-boot),
[Linux Kernel](https://www.kernel.org/)) but at some point you detect that the latest features of your CPU are not yet
upstream and therefore you need the changes done by the silicon vendor
(example NXP: [U-Boot](https://source.codeaurora.org/external/imx/uboot-imx/),
[Linux Kernel](https://source.codeaurora.org/external/imx/linux-imx/)) that get further refined and integrated by the
SOM/device vendor (example Variscite: [U-Boot](https://github.com/varigit/uboot-imx),
[Linux Kernel](https://github.com/varigit/linux-imx)).
Obviously you need to cross compile those components for your target system and for Debian I recommend packaging the
Kernel as a Debian package. Other than this the kernel does usually not differ between a Debian based system, and a
system that got built using any other solution.

Verdict: The bootloader and the kernel does not make the real difference between Debian and any other solution.

Root File System
----------------

A Debian root file system is built from Debian packages. The majority of the packages that go into an embedded Debian
system can directly be pulled as binary packages from an official Debian repository. Did you know? The official Debian
repositories do also host the corresponding source code. If needed, an official Debian package can also be tweaked and
then stored on a private Debian repository. Furthermore, you will write your own applications, and I strongly recommend
that you also make them available as Debian packages using your private Debian repository.

After all you will end up with a setup that might look like this:

![Image Handling](/assets/images/blog/apt_repository.png){:class="img-responsive"}

Both, the developer and the build server will be able to build target system images without having the need to access
a target device. The process to build the root file system for the target device is finally highly reproducible, does
fit perfectly into a modern CI/CD pipeline and does easily scale up to big teams.

Verdict: The root file system of an embedded Debian system can be tweaked where needed, and the usage of prebuilt binary
packages does speed up the workflow. As you do not only have the Debian packages available for the architecture of your
target system (e.g. arm64) but also for the architecture of your development machine (e.g. amd64) you can easily build
a digital twin of your target system and run it natively as a LXD container on your host system. This has proven to be
extremely convenient for developers.

Tools
-----

One hidden gem for embedded device development is the cross compiler. Some twenty years ago it felt like black magic
to get such a cross compiler compiled. Nowadays installing a
[cross compiler on Debian](https://packages.debian.org/bullseye/crossbuild-essential-arm64) is as easy as
switching on your computer: `sudo apt install crossbuild-essential-arm64`. Done!

The story does not end here: To compile an application for your target system you also need the corresponding
development libraries and this is where Debian has another shiny feature:
[multiarch](https://wiki.debian.org/Multiarch/HOWTO). Multiarch lets you install "foreign" libraries (e.g. arm64)
alongside the libraries of your host system (e.g. amd64). Now you are ready to cross compile complex applications.

I strongly recommend the usage of containers so that your development environment does correspond to your target system:

![Container Setup](/assets/images/blog/container-cycle.png){:class="img-responsive"}

So we have tackled the cross compilation - now let's move on to the image build: The only thing that the Debian
embedded community could be blamed for is that there are (too) many options:  

- [Elbe](https://elbe-rfs.org/): Debian based E.mbedded L.inux B.uild E.nvironment.
- [Apertis](https://www.apertis.org/): Collaborative OS platform for products.
- [ISAR](http://github.com/ilbers/isar): Integration System for Automated Root filesystem generation. Isar is also used
  by [Siemens Omni OS](https://www.plm.automation.siemens.com/global/en/products/embedded/omni-os.html).
- [edi](https://www.get-edi.io/): Embedded Development Infrastructure.
  My favourite because of [Ansible](https://www.ansible.com/) and the digital twin approach - but honestly, I am biased.

As described above, your Debian setup will require a Debian repository. Here are my recommendations:

- [packagecloud](https://packagecloud.io/): A reliable hosted solution for Debian package management.
- [Aptly](https://www.aptly.info/): The swiss army knife for self hosted Debian package management.
- [Artifactory](https://jfrog.com/artifactory): Artifactory can be hosted on premise, or you can opt for a cloud instance
  subscription.

Finally, here are some other tools and services that I would like to mention:

- [snapshot.debian.org](https://snapshot.debian.org/): "The snapshot archive is a wayback machine that allows access to
  old packages based on dates and version numbers."
- [Mender](https://mender.io/): Secure and reliable over-the-air updates for embedded systems. Mender fits perfectly
  into the [Debian ecosystem](https://packages.debian.org/bullseye/mender-client).
- [Reproducible Builds](https://tests.reproducible-builds.org/debian/reproducible.html): As an embedded developer you
  aim for a system that is highly reproducible.

Verdict: The tooling for embedded Debian is great and continues to become even better. The only thing you might struggle
with is the wide choice - but as a Linux developer you are used to it - aren't you?

Support
-------

If you once built embedded systems for highly regulated medical devices then you know the painful paperwork process
when doing a major version upgrade. You absolutely want to avoid this. At the same time you also need to fulfill the
cybersecurity requirements and therefore you need targeted security updates.

This turns out to be a unique selling point of Debian - you get long term support:

![Longevity](/assets/images/blog/debian_lts.png){:class="img-responsive"}

If this long term support is beneficial for your project too, then you should join the 
[Debian Long Term Support](https://www.freexian.com/en/services/debian-lts.html) initiative and also the
[Civil Infrastructure Platform](https://www.cip-project.org/) project might be of interest.

If you need personalized help, I am sure you will get good support from companies like
[Linutronix](https://linutronix.de/en/dienstleistung/ide_build-tools_elbe-debian.php),
[Ilbers](http://www.ilbers.de/en/isar.html) or
[Siemens (formerly Mentor Graphics)](https://www.plm.automation.siemens.com/global/en/products/embedded/omni-os.html).

Verdict: [Debian](https://www.debian.org/) has a support ecosystem that has attracted many big players over the past
few years. The longevity has proven to be a key selling point.

Conclusion
----------

All the above examples clearly show that you can go far beyond the "fast prototype" using Debian. Alone
[edi](https://www.get-edi.io) is helping to build images for an amazing number of embedded devices that are used all
around the world. Given that you have
a clean setup your Debian image builds are reproducible. The cross compilation couldn't be much smoother, and the long
term support has become a key selling point. The resulting images might be a bit bigger than when built with other
approaches. In any case it is also possible to generate them for a foreign platform - this is what I do on a daily basis.

I am very happy how Debian has evolved over the past few years:
**Debian for embedded systems - seriously!**
