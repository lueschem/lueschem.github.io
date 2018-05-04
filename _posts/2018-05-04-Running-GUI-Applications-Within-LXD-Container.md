---
author: matthias_luescher
author_profile: true
description: "In this blog post we extend our Raspbian cross compilation LXD container and add an integrated development environment (IDE) for C/C++."
comments: true
---

If you have read my previous blog posts about using containers for embedded development
I have explained most of the things using command line commands. I hope that this
convinced you that you can become a real nerd and do everything from the command line.
The fancy integrated development environments (IDEs) are anyway just something for
beginners! No?!

Ok, then you should read on. Honestly, I am also a huge fan of good IDEs and in this
blog post I will show how you can add your favorite IDE to your
[Raspbian cross compilation LXD container](/Cross-Compiling-for-Raspbian/).

## Prerequisites

The instructions below make use of the `gpu device` that got introduced with LXD 2.7.
Therefore I recommend that you use **Ubuntu Bionic Beaver (18.04 LTS)** as your host operating
system or that you upgrade your LXD installation at your own risk if you decide to stay
on Ubuntu Xenial Xerus (16.04 LTS) for some more time.

The following instructions have been tested on **Ubuntu 18.04** and they assume that
you already have a privileged container named **raspbian-stretch-cross** up and running
that got created according to the
[instructions contained in this blog post](/Cross-Compiling-for-Raspbian/).

## Preparing GPU Passthrough

The following steps need to be done once per container and they are executed on the host
system:

First we make the X server socket available within the container:

``` bash
lxc config device add raspbian-stretch-cross xorg-socket disk path=/tmp/.X11-unix/X0 source=/tmp/.X11-unix/X0
```

Then we share the `.Xauthority` file - which is needed for X server authentication -
into the container:

``` bash
lxc config device add raspbian-stretch-cross ${USER}-Xauthority disk path=/home/${USER}/.Xauthority source=${XAUTHORITY}
```

And finally we make some GPU specific files available within the container:

``` bash
lxc config device add raspbian-stretch-cross host-gpu gpu uid=0 gid=44
```

We are already done with the preparation!

## Running QtCreator From Within the Container

First, we have to find out the IPv4 address (`CONTAINER_IP`) of the container:

``` bash
lxc list -c n4 raspbian-stretch-cross
```

Now it is time to enter the container:

``` bash
ssh CONTAINER_IP
```

One of my favorite IDEs for C/C++ development is
[Qt Creator](https://www.qt.io/qt-features-libraries-apis-tools-and-ide/#ide).
We can directly install it from the Debian repository:

``` bash
sudo apt install qtcreator
```

And launch it after having set the `DISPLAY` environment variable:

``` bash
export DISPLAY=:0
qtcreator
```

On a high dpi screen you might want to adjust some scaling factors prior
to starting Qt Creator:

``` bash
export DISPLAY=:0
export QT_SCALE_FACTOR=1
export QT_AUTO_SCREEN_SCALE_FACTOR=0
export QT_SCREEN_SCALE_FACTORS=2
qtcreator
```

If everything went fine, the Qt Creator that you have launched within
the Debian stretch container should seamlessly appear on your Ubuntu
desktop:

![Qt Creator](/assets/images/blog/QtCreator.png){:class="img-responsive"}

## Adding Convenience

## Conclusion and Acknowledgement

