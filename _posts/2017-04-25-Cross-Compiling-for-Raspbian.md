---
author: matthias_luescher
author_profile: true
description: "Debian multiarch can be leveraged for speeding up the compilation of software for the Raspberry Pi."
comments: true
---

![raspberry](/assets/images/blog/raspberry.png){:class="img-responsive"}

As promised in the [previous blog post](/Compiling-for-Embedded-Debian-Target-Systems/) I will outline how we can leverage 
[multiarch](https://wiki.debian.org/Multiarch/HOWTO) in order to speed up the compilation process for the 
[Raspberry Pi]((http://www.raspberrypi.org)).

[Raspbian](https://www.raspbian.org/) is mainly a recompiled Debian. The recompilation was necessary because the official 
Debian armhf port requires an ARMv7-A capable CPU while the Pi 1 and Pi Zero are only equipped with an ARMv6 capable CPU. 

I rather prefer to automate things instead of writing lengthy instructions and therefore a few steps will be sufficient
to start the cross compilation adventure:

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

That's it - we are ready to compile a program:

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

And we compile it using the C++ cross compiler:

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
cd ..
mkdir -p openssl-cross && cd openssl-cross
apt-get source libssl1.0.0
cd openssl-1.0.1t
```

And we compile it:

``` bash
export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf
```

As expected, this was seven times faster than on the Raspberry Pi 3 Model B.

Now it is really time for the risky part of the game. We copy this library to the Raspberry Pi using ssh and we hope that the
subsequent installation will not brick our ssh access:

``` bash
scp ../libssl1.0.0_1.0.1t-1+deb8u6_armhf.deb pi@raspberrypi:
ssh pi@raspberrypi
sudo dpkg -i libssl1.0.0_1.0.1t-1+deb8u6_armhf.deb
```

You have been warned - I really bricked my ssh access once when I tried a compilation without the adjusted compiler settings.

Let us see wheter we have more luck this time:

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



apt-get source man-db

lueschem@raspbian-cross-01:~/edi-workspace$ sudo apt-get build-dep -a armhf man-db
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages have unmet dependencies:
 zlib1g-dev:armhf : Depends: zlib1g:armhf (= 1:1.2.8.dfsg-2) but it is not going to be installed
E: Build-dependencies for man-db could not be satisfied.

lueschem@raspbian-cross-01:~/edi-workspace$ apt-cache policy zlib1g
zlib1g:
  Installed: 1:1.2.8.dfsg-2+b1
  Candidate: 1:1.2.8.dfsg-2+b1
  Version table:
 *** 1:1.2.8.dfsg-2+b1 0
        500 http://httpredir.debian.org/debian/ jessie/main amd64 Packages
        100 /var/lib/dpkg/status
lueschem@raspbian-cross-01:~/edi-workspace$

apt-get source zlib1g

cd zlib-1.2.8.dfsg/

dch --bin-nmu --distribution unstable

export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf -B

sudo dpkg -i ../zlib1g_1.2.8.dfsg-2+b1_armhf.deb

sudo dpkg -i ../zlib1g-dev_1.2.8.dfsg-2+b1_armhf.deb

sudo apt-get build-dep -a armhf man-db

cd ../man-db-2.7.0.2/

debuild -us -uc -aarmhf

scp ../man-db_2.7.0.2-5_armhf.deb pi@raspberrypi:

ssh pi@raspberrypi

sudo dpkg -i man-db_2.7.0.2-5_armhf.deb

compile and install kernel
--------------------------

explanations
------------

link to tagged files of edi-raspbian



conclusion
----------

Not everything is yet as sweet as the raspberry pictured above. ...
