---
author: matthias_luescher
author_profile: true
description: "Debian is a great choice for embedded projects. This blog post presents three approaches for getting your source code compiled for the target system."
comments: true
---

Recently I bought a [Raspberry Pi](http://www.raspberrypi.org) 3 Model B since I wanted to evaluate the different strategies of 
compiling source code for such an ARM powered device.

The probably most popular operating system for the Raspberry Pi is [Raspbian](https://www.raspbian.org/). From my point of view this is
a very reasonable choice and therefore I prepared a flash card according to 
[this instructions](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md).

Since my two sons (4 and 6 years old) helped me to unpack and assemble everything it was now their turn to watch a "bob the builder" 
on this brand new "computer".

For this reason I had no access to the Raspberry Pi and this gave me some time to think about a good benchmark.

I decided that I want to recompile and repackage [openssl](https://www.openssl.org/). According to 
[sloccount](https://www.dwheeler.com/sloccount/) it consists 
of about 310k lines of C and some 4k lines of C++ code. Furthermore it also contains perl, assembler, shell and lisp code.

Due to the above mentioned shortage of Raspberry Pi's I opted for a test run on my notebook 
(equipped with [Ubuntu 16.04](http://releases.ubuntu.com/16.04/), amd64). The Raspbian that 
I have installed to the "bob the builder" device is based on [Debian jessie](https://www.debian.org/releases/jessie/). 
Therefore I decided to to build a Debian jessie LXC container for my first experiments:

``` bash
mkdir debian-amd64
cd debian-amd64
edi config init debian-jessie-amd64 debian-jessie-amd64
sudo edi -v lxc configure debian-jessie-amd64 debian-jessie-amd64-develop.yml
```

For the next few steps we will enter the container (password is _ChangeMe!_, please check 
[this documentation](https://docs.get-edi.io/en/latest/getting_started.html) 
if you are interested in more details) and install some additional software:

``` bash
lxc exec debian-jessie-amd64 -- login $USER
sudo apt install dpkg-dev devscripts m4 bc vim
```

Now it is time to adjust ```/etc/apt/sources.list``` within the container by adding a line such as
```deb-src http://ftp.ch.debian.org/debian/ jessie main```.
 
The following commands will fetch the source code of openssl for the current Debian installation (in our case openssl-1.0.1t):

``` bash
sudo apt update
cd edi-workspace
apt-get source libssl1.0.0
cd openssl-1.0.1t
```

Finally it is time to start the Debian package build:

``` bash
time debuild -us -uc
```

It took slightly less than **5 minutes** to get everything done on my notebook that is powered by an i7-7500U (Kaby Lake) CPU and the
result were two amd64 Debian packages that are obviously not installable on the Raspberry Pi due to its ARM architecture.

## Native Compilation on the Raspberry Pi

As the sun was shining I had an easy game to convince my sons to stop watching "bob the builder" and go out instead. Now the
Raspberry Pi shortage was resolved and I reproduced the above steps on the Raspberry Pi.
 
After the command

``` bash
time debuild -us -uc
```

I joined my sons outside.

Later in the evening I had a look at the result: The job was done after about **35 minutes** resulting in two Raspbian compatible
Debian packages. Honestly, I was surprised that it did not take even longer. The ARM Cortex-A53 CPU of the Raspberry Pi 3 Model B 
seems to be quite powerful.
 
The fact that the compilation on my notebook is seven times faster than on the Raspberry Pi made me think about an additional
compiling approach:

## Native Compilation within an Emulated Container

This time we will build an [armhf](https://wiki.debian.org/ArmHardFloatPort) Raspbian container that runs on the Intel notebook 
thanks to [QEMU](http://wiki.qemu.org):

``` bash
mkdir raspbian-armhf
cd raspbian-armhf
edi config init raspbian-jessie-armhf raspbian-jessie-armhf
sudo edi -v lxc configure raspbian-jessie-armhf raspbian-jessie-armhf-develop.yml
```

The setup of the _raspbian-jessie-armhf_ container takes considerably longer than for the _debian-jessie-amd64_ container 
due to the emulation.

After the successful creation we reproduce the same steps as for the _debian-jessie-amd64_ container.

Finally we run the build within this container:

``` bash
time debuild -us -uc
```

As expected the build produced two Raspbian compatible Debian packages. However - to my big disappointment - it took
more than **65 minutes**!

It was now time to scratch my head and remember my past when I was working with [cross compilers](https://en.wikipedia.org/wiki/Cross_compiler):

## Cross Compilation Within a Native Container

We return into our _debian_jessie_amd64_ container and pimp it with a cross compiler. As of Debian jessie cross compilers
are not part of the main repository and therefore we have to activate an additional repository:

First we add an additional line to our ```/etc/apt/sources.list``` which reads as follows: 
```deb http://emdebian.org/tools/debian/ jessie main```. Then we install the required archive key:

```
curl http://emdebian.org/tools/debian/emdebian-toolchain-archive.key | sudo apt-key add -
```

The security aware reader will notice that we should actually fetch such keys from a _https_ source.

Many people have done a great job that will allow us to proceed with the next step: A Debian based installation is aware of 
[multiple architectures](https://wiki.debian.org/Multiarch/HOWTO) and therefore we can enable armhf as a foreign architecture:
 
``` bash
sudo dpkg --add-architecture armhf
```

This will allow us to install armhf libraries alongside the amd64 libraries.

It is now time to update the apt cache:

``` bash
sudo apt update
```

Finally we are ready to install the [gcc](https://gcc.gnu.org/) based cross toolchain:

``` bash
sudo apt install crossbuild-essential-armhf
```

Again we want to build openssl and therefore we re-fetch the source code:

``` bash
cd edi-workspace
mv openssl-1.0.1t openssl-1.0.1t.old
apt-get source libssl1.0.0
cd openssl-1.0.1t
```

And anxiously waiting for the result we start the cross compilation:

``` bash
export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf
```

Great - only about **5 minutes**! This build was seven times faster than on the Raspberry Pi 3 Model B! 

Unfortunately it is too early
to celebrate. The Raspbian binaries are built with slightly different compiler options than those we have just been applying right now.
**Therefore do not install those packages on your Raspbian installation!** 

Bob would now ask: "Can we fix it?" And the whole team knows the answer: "Yes we can!"

I will show the necessary steps in my [next blog post](/Cross-Compiling-for-Raspbian/)!


## Conclusion

Cross compilation can speed up your development cycles for embedded devices. The Debian/Ubuntu community has done a great job by
introducing [multiarch](https://wiki.debian.org/Multiarch/HOWTO) within recent versions of their distributions. It makes cross
compiling very sleek.

If you stumble upon a piece of software that persistently refuses to get cross compiled - unfortunately this might happen - you are still
able to compile it within an emulated container or on the target hardware.

While cross compilation is typically done within a [chroot](https://en.wikipedia.org/wiki/Chroot), we use LXC/LXD containers instead.
The overhead introduced by LXC/LXD is negligible and the added value is huge if you are planning to do additional stuff within the
container. Please check [Stéphane Graber's website](https://stgraber.org/2016/03/11/lxd-2-0-blog-post-series-012/) 
if you want to learn more about LXC/LXD.




