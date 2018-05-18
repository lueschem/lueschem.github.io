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
to start the cross compilation adventure on an Ubuntu (16.04 or newer) host:

*Update - 26-March-2018*
------------------------

_Due to the great demand, I finally upgraded the setup to Raspbian stretch! You can now enjoy gcc 6.3 within the
cross compilation container. The instructions below do now reflect this version change._


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
sudo edi -v lxc configure raspbian-stretch-cross stretch-cross.yml
```

And enter it (password is _ChangeMe!_, you can change it using ```passwd```):

``` bash
lxc exec raspbian-stretch-cross -- login ${USER}
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
indeed an armhf binary: ```file hello-raspbian```. If you are still thinking that this executable is not trustworthy
then you are welcome to download it to your Raspberry Pi and execute it there.

If you really want to know why we were just able to run this armhf binary within the amd64 container then you can run
the following command:

``` bash
 strace ./hello-raspbian 2>&1 | grep qemu-arm-static
```

Maybe we are now curious where we got this shiny cross compiler from. The following commands

``` bash
dpkg -S $(realpath $(which arm-linux-gnueabihf-g++))
apt-cache policy g++-6-arm-linux-gnueabihf
```

will reveal that the cross compiler got downloaded from a 
[custom repository](https://get-edi.github.io/raspbian-cross-compiler/).
The reason for having this custom repository is that I had to recompile some Debian binaries in order to garantee
correct cross compilation with the options ```-march=armv6 -mfpu=vfp``` instead of
```-march=armv7-a -mfpu=vfpv3-d16 -mthumb```.

Obviously, a hello world program is boring and we will now do something that will break our Raspberry Pi ssh access if we
should have done something wrong with the cross toolchain:

Recompilation of the SSL Library
--------------------------------

Like within the [previous blog post](/Compiling-for-Embedded-Debian-Target-Systems/) we fetch the source code:

``` bash
mkdir -p ~/edi-workspace/openssl-cross && cd ~/edi-workspace/openssl-cross
apt-get source libssl1.1
cd openssl-1.1.0f
```

And we compile it:

``` bash
export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf
```

As expected, this was seven times faster than on the Raspberry Pi 3 Model B.

Now it is time for the risky part of the game. We copy this library to the Raspberry Pi using ssh and we hope that the
subsequent installation will not brick our ssh access:

``` bash
scp ../libssl1.1_1.1.0f-3+deb9u1_armhf.deb pi@RASPBERRY_PI_IP_ADDRESS:
ssh pi@RASPBERRY_PI_IP_ADDRESS
sudo dpkg -i libssl1.1_1.1.0f-3+deb9u1_armhf.deb
```

You have been warned - I really bricked my ssh access once when I tried to install binaries that were compiled
without the adjusted compiler settings.

Let us see whether we have more luck this time:

```
sudo systemctl restart ssh
exit
ssh pi@RASPBERRY_PI_IP_ADDRESS
exit
```

Uff - everything went well!

The following example will show us the cross compilation of a package with slightly more dependencies:

Recompilation of man-db
-----------------------

First we fetch the source code of man-db (the tools that deal with man pages). This was a truly random choice of a package
that contains executables. Within the container _raspbian-jessie-cross_ we execute the following commands:

``` bash
cd ~/edi-workspace
apt-get source man-db
cd man-db-2.7.6.1 
```

man-db is with regards to dependencies slightly more complex than the openssl library and therefore we need some build dependencies:

``` bash
sudo apt-get build-dep -a armhf man-db
```

Now we have green lights for the man-db build:

``` bash
debuild -us -uc -aarmhf
```

Out of curiosity we can install this package on the Raspberry Pi:

``` bash
scp ../man-db_2.7.6.1-2_armhf.deb pi@RASPBERRY_PI_IP_ADDRESS:
ssh pi@RASPBERRY_PI_IP_ADDRESS
sudo dpkg -i man-db_2.7.6.1-2_armhf.deb
```

With the commmand ```man man``` we can verify that the cross compiled binary seems to work fine on the Raspberry Pi.

Conclusion
----------

With the above description only a few steps are needed to prepare a cross development toolchain for Raspbian. The tricky 
part of the setup is hidden behind an [Ansible](https://www.ansible.com/) playbook that is kicked off by the tool
[edi](http://www.get-edi.io).
If you are interested in the details you can take a look at the playbook that is part of the 
[edi-raspbian git repository](https://github.com/lueschem/edi-raspbian).

With the very same toolchain I have also successfully compiled a recent Raspberry Pi kernel.

Everything above is still new and experimental and I do not expect it to work under all circumstances. I am sure that there
are more knowledgeable people around than me who can improve the above setup. Pull requests or constructive comments are
highly appreciated!

Further Reading
---------------

Please read this [blog post](/A-new-Approach-to-Operating-System-Image-Generation/) if you are interested in running
pure Debian on a Raspberry Pi 2 or 3.

If you would like to add your favourite integrated development environment (IDE) to your cross compilation
toolchain you can read on [here](/Running-GUI-Applications-Within-LXD-Container/).
