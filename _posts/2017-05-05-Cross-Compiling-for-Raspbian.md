---
author: matthias_luescher
author_profile: true
description: "Debian multiarch can be leveraged for speeding up the compilation of software for the Raspberry Pi."
comments: true
---

As promised in the [previous blog post](/Compiling-for-Embedded-Debian-Target-Systems/) I will outline how we can leverage 
[multiarch](https://wiki.debian.org/Multiarch/HOWTO) in order to speed up the compilation process for the 
[Raspberry Pi](http://www.raspberrypi.org).

![raspberry](/assets/images/blog/raspberry.png){:class="img-responsive"}

[Raspbian](https://www.raspbian.org/) is mainly a recompiled Debian. The recompilation was necessary because the official 
Debian armhf port requires an ARMv7-A capable CPU while the Pi 1 and Pi Zero are only equipped with an ARMv6 capable CPU. 

I rather prefer to automate things instead of writing lengthy instructions and therefore a few steps will be sufficient
to start the cross compilation adventure on an Ubuntu 16.04 host:

*Update - 28-November-2017*
---------------------------

_The setup described in this blog post remains valid for Raspbian jessie. However Raspbian is now also available based
on Debian stretch. On Debian stretch gcc is treated differently than on jessie and it makes it more difficult to mix
Raspbian stretch armhf binaries with pure Debian stretch amd64 binaries. Therefore I am probably not going to update
this setup for Raspbian stretch._

_Luckily, the community has made huge headway in running pure Debian on the Raspberry Pi 2 and 3. This is why I started
a new project called [edi-pi](https://github.com/lueschem/edi-pi). With edi-pi you can easily create the following
artifacts:_

* _A pure Debian stretch arm64 image for the Raspberry Pi 3._
* _An amd64/arm64 based LXD container with a pre-installed cross development toolchain for C and C++._
* _An emulated arm64 LXD container._

_Please read this [blog post](/A-new-Approach-to-Operating-System-Image-Generation/) for more details._

Container Setup
---------------

The following repository contains the container setup instructions:

``` bash
git clone https://github.com/lueschem/edi-raspbian.git
cd edi-raspbian
```

Under the assumption that [edi is already installed](http://docs.get-edi.io/en/latest/getting_started.html), we can directly
generate our mulitarch cross compilation container:

``` bash
sudo edi -v lxc configure raspbian-jessie-cross jessie-cross.yml
```

And enter it (password is _ChangeMe!_, you can change it using ```passwd```):

``` bash
lxc exec raspbian-jessie-cross -- login ${USER}
```

That's it - we are ready to cross compile a program:

Hello World
-----------

We change into the shared workspace folder _edi-workspace_ and within the subfolder _hello_ we write some code:

```
mkdir -p ~/edi-workspace/hello
cd ~/edi-workspace/hello
cat << EOF > hello.cpp
#include <iostream>
 
int main()
{
    std::cout << "Hello Raspbian!" << std::endl;
    return 0;
}
EOF
```

And we compile it using the pre installed C++ cross compiler:

``` bash
arm-linux-gnueabihf-g++ -v hello.cpp -o hello-raspbian
```

And obviously - we want to test our brand new executable:

``` bash
./hello-raspbian
```

But wait - why were we able to run this binary? Just in case we do not trust the cross compiler we can verify that it is 
indeed an armhf binary: ```file hello-raspbian```. If you are still thinking that this executable is not trustworthy then you are
welcome to download it to your Raspberry Pi and execute it there.

If you really want to know why we were just able to run this armhf binary within the amd64 container then you can run the following command:

``` bash
 strace ./hello-raspbian 2>&1 | grep qemu-arm-static
```

Maybe we are now curious where we got this shiny cross compiler from. The following commands

``` bash
dpkg -S $(which arm-linux-gnueabihf-g++)
apt-cache policy g++-arm-linux-gnueabihf
```

will reveal that the cross compiler got downloaded from the [emdebian jessie tools repository](http://emdebian.org/tools/debian/).
You might now wonder why this compiler was choosing the right options for Raspbian - namely 
```-march=armv6 -mfpu=vfp``` instead of ```-march=armv7-a -mfpu=vfpv3-d16 -mthumb```. This adjustment
was brought in through an adapted [gcc specs file](https://gcc.gnu.org/onlinedocs/gcc/Spec-Files.html).

Obviously, a hello world program is boring and we will now do something that will break our Raspberry Pi ssh access if we
should have done something wrong with the cross toolchain:

Recompilation of the SSL Library
--------------------------------

Like within the [previous blog post](/Compiling-for-Embedded-Debian-Target-Systems/) we fetch the source code:

``` bash
mkdir -p ~/edi-workspace/openssl-cross && cd ~/edi-workspace/openssl-cross
apt-get source libssl1.0.0
cd openssl-1.0.1t
```

And we compile it:

``` bash
export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf
```

As expected, this was seven times faster than on the Raspberry Pi 3 Model B.

Now it is time for the risky part of the game. We copy this library to the Raspberry Pi using ssh and we hope that the
subsequent installation will not brick our ssh access:

``` bash
scp ../libssl1.0.0_1.0.1t-1+deb8u6_armhf.deb pi@raspberrypi:
ssh pi@raspberrypi
sudo dpkg -i libssl1.0.0_1.0.1t-1+deb8u6_armhf.deb
```

You have been warned - I really bricked my ssh access once when I tried to install binaries that were compiled
without the adjusted compiler settings.

Let us see whether we have more luck this time:

```
sudo systemctl restart ssh
exit
ssh pi@raspberrypi
exit
```

Uff - everything went well!

The following example will show us some rough edges that we might hit here and there:

Recompilation of man-db
-----------------------

First we fetch the source code of man-db (the tools that deal with man pages). This was a truly random choice of a package
that contains executables. Within the container _raspbian-jessie-cross_ we execute the following commands:

``` bash
cd ~/edi-workspace
apt-get source man-db
```

man-db is with regards to dependencies slightly more complex than the openssl library and therefore we need some build dependencies:

``` bash
sudo apt-get build-dep -a armhf man-db
```

Unfortunately this fails:

``` bash
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages have unmet dependencies:
 zlib1g-dev:armhf : Depends: zlib1g:armhf (= 1:1.2.8.dfsg-2) but it is not going to be installed
E: Build-dependencies for man-db could not be satisfied.
```

Some further investigation using ```apt-cache policy zlib1g``` reveals

``` bash
zlib1g:
  Installed: 1:1.2.8.dfsg-2+b1
  Candidate: 1:1.2.8.dfsg-2+b1
  Version table:
 *** 1:1.2.8.dfsg-2+b1 0
        500 http://httpredir.debian.org/debian/ jessie/main amd64 Packages
        100 /var/lib/dpkg/status
```

that the armhf package version (1:1.2.8.dfsg-2) does not fully match the amd64 package version (1:1.2.8.dfsg-2+b1)
and therefore apt refuses to do the installation. Please do not blame Debian for this outcome since it was my decision to mix Debian 
amd64 packages with Raspbian armhf packages. After a closer investigation of the version difference I learned something new about
package versions: The ```+b1``` version suffix indicates that [a binary only rebuild (binNMU)](https://wiki.debian.org/binNMU) was triggered 
on the amd64 platform for the zlib1g package. Debian seems to synchronize such binNMU rebuilds over all architectures and therefore we
would not hit the above problem on a pure Debian multiarch installation.

Luckily, in our case the workaround is quite simple. We will reproduce the binNMU rebuild for our Raspbian package:
 
``` bash
apt-get source zlib1g
cd zlib-1.2.8.dfsg/
dch --bin-nmu --distribution unstable ""
export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf -B
```

And install the resulting packages:

``` bash
sudo dpkg -i ../zlib1g_1.2.8.dfsg-2+b1_armhf.deb
sudo dpkg -i ../zlib1g-dev_1.2.8.dfsg-2+b1_armhf.deb
```

Now we have green lights for the man-db build:

``` bash
sudo apt-get build-dep -a armhf man-db
cd ../man-db-2.7.0.2/
debuild -us -uc -aarmhf
```

Out of curiosity we can install this package on the Raspberry Pi:

``` bash
scp ../man-db_2.7.0.2-5_armhf.deb pi@raspberrypi:
ssh pi@raspberrypi
sudo dpkg -i man-db_2.7.0.2-5_armhf.deb
```

With the commmand ```man man``` we can verify that the cross compiled binary seems to work fine on the Raspberry Pi.

Conclusion
----------

With the above description only a few steps are needed to prepare a cross development toolchain for Raspbian. The tricky 
part of the setup is hidden behind an [Ansible](https://www.ansible.com/) playbook that is kicked off by the tool [edi](http://www.get-edi.io).
If you are interested in the details you can take a look at the playbook that is part of the 
[edi-raspbian git repository](https://github.com/lueschem/edi-raspbian).

With the very same toolchain I have also successfully compiled a recent Raspberry Pi kernel. The subsequent installation went fine
and the Raspberry Pi is now up since 11 days.

However, not everything is yet as sweet as the raspberry pictured above. Just because it was the benchmark in
[a Raspberry forum thread](https://www.raspberrypi.org/forums/viewtopic.php?f=66&t=17254), I also tried to compile 
[Pidgin](https://pidgin.im/). After some desperate attempts to get all build dependencies installed, I threw in the towel.

Everything above is still new and experimental and I do not expect it to work under all circumstances. I am sure that there
are more knowledgeable people around than me who can improve the above setup. Pull requests or constructive comments are
highly appreciated!