---
author: matthias_luescher
author_profile: true
description: "Debian multiarch can be leveraged for speeding up the compilation of software for the Raspberry Pi. This blog post describes how you can setup a cross compilation environment for Raspbian buster."
comments: true
title: "Cross Compiling for Raspbian Buster"
---

Guessing from the number of clones on GitHub my [cross compilation setup](https://github.com/lueschem/edi-raspbian)
for Raspbian stretch still seems to be pretty popular. This makes it worthwhile to move on to
Raspbian buster and enjoy the gcc update (the version jumped from 6.3 to 8.3).

![raspberry](/assets/images/blog/raspberry.png){:class="img-responsive"}

This post serves as an addendum to my [previous post](/Cross-Compiling-for-Raspbian/) covering the same topic.

Container Setup
---------------

As a first step we build a cross compilation container (Debian buster amd64) that contains a cross compiler that
knows how to build binaries for the Raspbian armhf platform:

The following [git repository](https://github.com/lueschem/edi-raspbian) contains the container setup instructions.
We clone it to our Ubuntu (>= 16.04) or Debian (>= stretch) development machine:

``` bash
git clone https://github.com/lueschem/edi-raspbian.git
cd edi-raspbian
```

Under the assumption that [edi is already installed](https://docs.get-edi.io/en/latest/getting_started.html), we can directly
generate our [mulitarch](https://wiki.debian.org/Multiarch/HOWTO) cross compilation container (please carefully setup
your ssh keys before starting the container build):

``` bash
sudo edi -v lxc configure raspbian-buster-cross buster-cross.yml
```

To retrieve the IP address of the container (`IP_ADDRESS_OF_CONTAINER`) we use the following command:

``` bash
lxc list raspbian-buster-cross
```

To add some convenience we insert the following snippet into `~/.ssh/config`:

``` bash
Host raspbian-buster-cross
    Hostname IP_ADDRESS_OF_CONTAINER
```

Now we can enter the container using ssh:

``` bash
ssh raspbian-buster-cross
```

The sudo password within the container is _ChangeMe!_, you can change it using ```passwd```.

Hello World
-----------

To explain a typical workflow we make use of the standard hello world example.

Within the container we change into the shared workspace folder _edi-workspace_ and within 
the sub-folder _hello_ we write some code:

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

To make the development cycle fast and convenient, we can already verify the correct behaviour
of our code without even touching the Raspberry Pi.

```
g++ hello.cpp -o hello-amd64
./hello-amd64
```

Now that we are sure that the code works on our development host, we can cross compile it for the Raspberry Pi:

``` bash
arm-linux-gnueabihf-g++ hello.cpp -o hello-raspbian
```

The resulting `hello-raspbian` binary can now be copied over to the Raspberry Pi and it should execute properly even
on a Raspberry Pi 1 that does not support ARMv7-A instructions (they would be required by Debian armhf).

Using CMake
-----------

Now we can turn our hello world example into a simple CMake project.

First we have to install CMake within the container:

``` bash
sudo apt install cmake
```

Now we write a minimal CMake configuration:

```
cat << EOF > CMakeLists.txt
cmake_minimum_required (VERSION 2.6)
project (hello)
add_executable(hello-raspbian-cmake hello.cpp)
EOF
```

To do the cross compilation we need a toolchain file:

```
cat << EOF > toolchain.armhf
SET (CMAKE_SYSTEM_NAME Linux)
SET (CMAKE_SYSTEM_PROCESSOR armhf)

SET (CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
SET (CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)

SET (CMAKE_FIND_ROOT_PATH /usr/arm-linux-gnueabihf)
SET (ONLY_CMAKE_FIND_ROOT_PATH TRUE)
SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

IF (CMAKE_CROSSCOMPILING)
  MESSAGE("CROSS COMPILING for ${CMAKE_C_COMPILER}")
  INCLUDE_DIRECTORIES(BEFORE ${CMAKE_FIND_ROOT_PATH}/include)
ENDIF (CMAKE_CROSSCOMPILING)
EOF
```

Finally we do the cross compilation using CMake:

```
cmake -DCMAKE_TOOLCHAIN_FILE=toolchain.armhf .
make
```

Just to make sure that we really cross compiled the binary we take a closer look at it:

```
$ file ./hello-raspbian-cmake 
./hello-raspbian-cmake: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=fae4f484bef40eb2f71a46e8e1018dd0d258f47d, not stripped
```

Note: This is an absolutely minimal CMake setup. A more advanced setup will allow you to build amd64 and armhf binaries
side by side.

Recompilation of the SSL Library
--------------------------------

As within the [previous blog post](/Cross-Compiling-for-Raspbian/) we can now cross-compile a Raspbian package:

``` bash
mkdir -p ~/edi-workspace/openssl-cross && cd ~/edi-workspace/openssl-cross
apt-get source libssl1.1
cd openssl-1.1.1d
export DEB_BUILD_OPTIONS=nocheck; debuild -us -uc -aarmhf
```

The resulting armhf Raspbian package can now be installed on a Raspberry Pi.

Dealing with +rpi1 Issues
-------------------------

In this mixed Debian/Raspbian environment you might encounter situations, where the package versions of Raspbian
do not exactly match the package versions of Debian. Let's assume that we want to recompile apt and therefore we
would like to install the required build dependencies:

``` bash
$ sudo apt build-dep -aarmhf apt
Reading package lists... Done
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 builddeps:apt:armhf : Depends: g++:armhf (>= 4:7) but it is not going to be installed
                       Depends: libsystemd-dev:armhf but it is not going to be installed
                       Depends: libudev-dev:armhf but it is not going to be installed
                       Depends: libzstd-dev:armhf (>= 1.0) but it is not going to be installed
E: Unable to correct problems, you have held broken packages.
```

Ouch - this did not work out as expected. The "-dev" packages require an exact version match:

``` bash
$ sudo apt install libzstd-dev:armhf
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 libzstd-dev:armhf : Depends: libzstd1:armhf (= 1.3.8+dfsg-3+rpi1) but it is not going to be installed
E: Unable to correct problems, you have held broken packages
```

From Raspbian we can get:

``` bash
$ apt-cache policy libzstd1:armhf
libzstd1:armhf:
  Installed: (none)
  Candidate: 1.3.8+dfsg-3+rpi1
  Version table:
     1.3.8+dfsg-3+rpi1 400
        400 http://mirrordirector.raspbian.org/raspbian buster/main armhf Packages
```

... and from Debian:

``` bash
$ apt-cache policy libzstd1      
libzstd1:
  Installed: 1.3.8+dfsg-3
  Candidate: 1.3.8+dfsg-3
  Version table:
 *** 1.3.8+dfsg-3 500
        500 http://deb.debian.org/debian buster/main amd64 Packages
        100 /var/lib/dpkg/status
```

In fact, the versions are slightly different (1.3.8+dfsg-3*+rpi1* versus 1.3.8+dfsg-3).

Luckily we can fix the issue by compiling libraries with matching versions. First we have to enable the Raspbian
sources within the container. To do so, we can add the following line

```
deb-src http://mirrordirector.raspbian.org/raspbian/ buster main contrib non-free rpi
```

to `/etc/apt/sources.list.d/raspbian_buster.list`.

Now we can recompile and install the missing libzstd1 library:

```
sudo apt update
sudo apt build-dep libzstd1
apt source libzstd1
cd libzstd-1.3.8+dfsg
debuild -us -uc
sudo dpkg -i ../libzstd1_1.3.8+dfsg-3+rpi1_amd64.deb ../libzstd-dev_1.3.8+dfsg-3+rpi1_amd64.deb
```

Finally we can also install the armhf development package:

```
sudo apt install libzstd-dev:armhf
```

The same steps are required for systemd:

```
sudo apt build-dep systemd
apt source systemd
cd systemd-241
debuild -us -uc
sudo dpkg -i ../libsystemd0_241-7~deb10u1+rpi1_amd64.deb ../libsystem
d-dev_241-7~deb10u1+rpi1_amd64.deb ../libudev1_241-7~deb10u1+rpi1_amd64.deb  ../libudev-dev_241-7~deb10u1+rpi1_amd64.deb
sudo dpkg -i ../systemd_241-7~deb10u1+rpi1_amd64.deb ../systemd-sysv_241-7~deb10u1+rpi1_amd64.deb
```

After some additional tricks I was able to cross compile the apt package.

Conclusion
----------

The cross compilation experience for Raspbian is not as smooth as for a
[pure Debian setup](https://github.com/lueschem/edi-pi) - but it works.

A kernel cross compilation is straight forward while more effort and expertise is required to cross compile non
trivial packages such as apt.

Furthermore the above generated container can be used as a "digital twin" of the Raspberry Pi and a lot of
development effort can happen within the container (without touching the target hardware).

If some components perfidiously refuse to get cross compiled, you can still buy the most powerful Raspberry Pi
available and natively compile them on that device. 

Further Reading
---------------

Please read this [blog post](/A-new-Approach-to-Operating-System-Image-Generation/) if you are interested in running
pure Debian on a Raspberry Pi 2 or 3.

If you would like to add your favourite integrated development environment (IDE) to your cross compilation
toolchain you can read on [here](/Running-GUI-Applications-Within-LXD-Container/).

You can also take a look at [the presentation]({{ site.url }}/assets/pdfs/DebianCross.pdf) I did for an
embedded GNU/Linux developer meet-up.
