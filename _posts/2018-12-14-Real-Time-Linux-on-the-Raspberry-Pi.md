---
author: matthias_luescher
author_profile: true
description: "This blog post outlines how you can build a PREEMPT_RT patched arm64 Linux kernel for the Raspberry Pi. Furthermore it gives some insights of the benefits that PREEMPT_RT delivers."
comments: true
title: "Real-Time Linux on the Raspberry Pi"
---

Some ten years ago I learned that it is possible to turn Linux into a real-time operating system
by applying the [PREEMPT_RT patch set](https://wiki.linuxfoundation.org/realtime/start).
I was more than fascinated by the opportunity of combining a general purpose operating system
with a real-time operating system and I immediately started to assess the maturity of the
approach. After a thorough investigation I finally concluded that PREEMPT_RT patched Linux
was a great match for the industrial machines that I was helping to build at that time. Today I am
very happy that a lot of other companies have chosen the same approach and that PREEMPT_RT patched
Linux has not only survived until today but that it has grown into a solution that is widely
used. I am also very grateful that the kernel developers have continued
[their journey](https://wiki.linuxfoundation.org/realtime/rtl/blog#guest-blog-post-from-bmw-car-itreal-time-linux-continues-its-way-to-main-line-development-and-beyond)
despite impediments.

Recently I did a [short presentation](https://www.get-edi.io/assets/pdfs/RealTimeLinux.pdf) 
about this topic and to give some more insights I compiled a PREEMPT_RT patched kernel for my
[Raspberry Pi 3 that is running arm64 Debian](/A-new-Approach-to-Operating-System-Image-Generation/).

## Building the Kernel

To speed up the kernel build the below steps are executed within a 
[cross compilation container](https://github.com/lueschem/edi-pi#creating-a-cross-development-lxd-container).

Therefore we enter the container first, install some required packages and prepare a work folder:

``` bash
ssh IP_OF_EDI_PI_CROSS_DEV
sudo apt install build-essential bc kmod cpio flex cpio libncurses5-dev bison libssl-dev wget
mkdir -p ~/edi-workspace/rt-kernel && cd ~/edi-workspace/rt-kernel
```

Now we download the kernel source and the matching PREEMPT_RT patch:

``` bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.19.8.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/4.19/older/patch-4.19.8-rt6.patch.xz
```

Finally we extract the source code and apply the PREEMPT_RT patch:

``` bash
tar xf linux-4.19.8.tar.xz
cd linux-4.19.8
xzcat ../patch-4.19.8-rt6.patch.xz | patch -p1
```

As a next step we grab a kernel configuration from the Raspberry Pi project and we do a
`make olddefconfig`:

``` bash
wget -O .config https://raw.githubusercontent.com/raspberrypi/linux/rpi-4.18.y/arch/arm64/configs/bcmrpi3_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
```

And now we want to switch on the feature "Fully Preemptible Kernel" (CONFIG_PREEMPT_RT_FULL):

``` bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```

Within the menu navigate to _General Setup/Preemption Model_ and choose _Fully Preemptible Kernel_.

Not related to PREEMPT_RT we need another modification:

Navigate back to the root menu by choosing _Exit_ and choose _Boot options_. Make sure that
the _Default kernel command string_ is completely empty. Choose _Exit_ twice and do not forget
to save the configuration.

Finally we compile the kernel and wrap it into a Debian package:

``` bash
make -j $(nproc) KBUILD_IMAGE=arch/arm64/boot/Image ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- deb-pkg
```

## Testing the Real-Time Kernel

Now we copy the resulting kernel from our Ubuntu host to the Raspberry Pi:

``` bash
scp ~/edi-workspace/rt-kernel/linux-image-4.19.8-rt6-v8_4.19.8-rt6-v8-1_arm64.deb pi@IP_OF_RASPBERRY:
```

For benchmarking the kernel we install the cyclictest executable on the Raspberry Pi
(default password is _raspberry_):

``` bash
ssh pi@IP_OF_RASPBERRY
sudo apt update && sudo apt install rt-tests
```

Before we install the real-time kernel we do a small benchmark:

``` bash
pi@raspberry:~$ sudo cyclictest -p 99 -t5 -n -i250
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 1.57 0.69 0.28 1/104 5233          

T: 0 ( 4835) P:99 I:250 C: 372513 Min:     14 Act:   27 Avg:   29 Max:    3009
T: 1 ( 4836) P:99 I:750 C: 124311 Min:     18 Act:   28 Avg:   29 Max:     590
T: 2 ( 4837) P:99 I:1250 C:  74550 Min:     18 Act:   22 Avg:   34 Max:    2803
T: 3 ( 4838) P:99 I:1750 C:  53268 Min:     20 Act:   32 Avg:   35 Max:    4747
T: 4 ( 4839) P:99 I:2250 C:  41436 Min:     22 Act:   31 Avg:   35 Max:    2255
```

Now we install the patched kernel, reboot ...

``` bash
sudo dpkg -i linux-image-4.19.8-rt6-v8_4.19.8-rt6-v8-1_arm64.deb
sudo reboot
sleep 60
ssh pi@IP_OF_RASPBERRY
```

... and repeat the benchmark:

``` bash
pi@raspberry:~$ sudo cyclictest -p 99 -t5 -n -i250
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 2.87 1.60 0.67 1/120 1315          

T: 0 (  603) P:99 I:250 C: 771448 Min:     17 Act:   33 Avg:   35 Max:     177
T: 1 (  604) P:99 I:750 C: 257149 Min:     18 Act:   35 Avg:   38 Max:     205
T: 2 (  605) P:99 I:1250 C: 154289 Min:     18 Act:   29 Avg:   35 Max:     145
T: 3 (  606) P:99 I:1750 C: 110206 Min:     18 Act:   29 Avg:   37 Max:     148
T: 4 (  607) P:99 I:2250 C:  85716 Min:     18 Act:   32 Avg:   36 Max:     164
```

As you can see, the PREEMPT_RT patch set brought down the maximum latency from about 5ms
to about 200us!

## Architectural Considerations

With a PREEMPT_RT patched Linux kernel you can build a system according to this picture:

![real-time Linux system](/assets/images/blog/rt_linux_setup.png){:class="img-responsive"}

The solution comes with the following design parameters:

- Real-time Linux can run on standard hardware.
- A GUI process can run on the same machine as the real-time process.
- The best supported CPU architecture is amd64.
- Real-time applications can run in user space.
- Plain C/C++ is suitable for real-time application development.
- With a [decent hardware](https://www.osadl.org/QA-Farm-Realtime.qa-farm-about.0.html)
you can operate a fieldbus with update cycles of up to 5kHz.
- Thanks to the CPU power of the Linux box and its real-time behavior
you can centrally control complex multi axis machines with relatively
dumb fieldbus controller nodes.
- Many processing tasks can be moved from the fieldbus node to the Linux box.

However, I strongly recommend to keep the real-time application slim as you can easily run into
troubles when trying to squeeze everything into one process with real-time priorities.

## Conclusion

PREEMPT_RT patched Linux is heavily used in industry as it is a brilliant combination of a
general purpose operating system with a real time operating system. In many cases it is a suitable
replacement for proprietary real-time operating systems.

It would be great if you could convince your company to sponsor the PREEMPT_RT development so
that it can finally fully slip into the Linux mainline kernel.
