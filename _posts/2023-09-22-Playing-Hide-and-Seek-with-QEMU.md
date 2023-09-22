---
author: matthias_luescher
author_profile: true
description: "QEMU is an indispensable tool for almost every developer that is dealing with embedded devices. Time to better understand it!"
comments: true
title: "Playing Hide and Seek with QEMU"
---

Have you ever wondered why you can run an arm64 Docker container on an amd64 host machine? Magic? 
Did you also run into the situation where most of the things were working but one arm64 application kept crashing?

Recently I was discussing the second question with a coworker and found out that my knowledge was outdated. Time to
catch up and learn more about the magic - [QEMU](https://www.qemu.org/) - that lets us run arm64 binaries on a host
machine that has an amd64 CPU.

In my case the relevant part is the [QEMU user space emulator](https://www.qemu.org/docs/master/user/main.html).
Within the [edi project](https://www.get-edi.io) I am using QEMU to build operating system images for a foreign
CPU architecture. When integrating a new Debian release I sometimes run into the situation where exactly one of the
crucial binaries crashes when running in an emulated environment. The culprit is usually an outdated QEMU emulator
and luckily I found a [convenient solution](https://docs.get-edi.io/en/latest/config_management/yaml.html?highlight=qemu#qemu-section)
to deal with it: edi just downloads and then copies a recent QEMU into the chroot or container and the case is closed.
This was working perfectly in the past.

# The Past - Ubuntu 18.04

Let's explore how it was working on Ubuntu 18.04!

We first install the required packages:

``` bash
ml@amd64-u1804:~$ sudo apt-get install debootstrap qemu-user-static
```

Then we bootstrap a minimal root file system using the
[two stage debootstrap approach](https://askubuntu.com/questions/287789/what-is-debootstrap-second-stage-for).

First stage:

``` bash
ml@amd64-u1804:~$ debootstrap_dir=buster-arm64
ml@amd64-u1804:~$ sudo debootstrap --arch arm64 --foreign buster ${debootstrap_dir} http://deb.debian.org/debian
```

Second stage:

``` bash
ml@amd64-u1804:~$ sudo chroot ${debootstrap_dir} /debootstrap/debootstrap --second-stage
chroot: failed to run command ‘/debootstrap/debootstrap’: No such file or directory
```

Failure!

But wait a second - '/debootstrap/debootstrap' is there:

``` bash
ml@amd64-u1804:~$ file ${debootstrap_dir}/debootstrap/debootstrap
buster-arm64/debootstrap/debootstrap: POSIX shell script, ASCII text executable
```

Let's chroot into the minimal root file system:

``` bash
ml@amd64-u1804:~$ sudo chroot ${debootstrap_dir} 
chroot: failed to run command ‘/bin/bash’: No such file or directory
```

Failure!

But the bash binary is there:

``` bash
ml@amd64-u1804:~$ file ${debootstrap_dir}/bin/bash
buster-arm64/bin/bash: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 3.7.0, BuildID[sha1]=b11533bde88bb45ef2891fbf3ad86c1869ed3a41, stripped
```

The thing is that it is an aarch64 binary and my Intel CPU has no clue what to do with it.

This is where QEMU comes to the rescue: Let's just copy QEMU into our minimal root file system (here we take the binary
from the host while edi offers the possibility to take a newer one):

``` bash
ml@amd64-u1804:~$ sudo cp $(which qemu-aarch64-static) ${debootstrap_dir}/usr/bin
```

Finally we are able to execute the second stage of debootstrap:

``` bash
ml@amd64-u1804:~$ sudo chroot ${debootstrap_dir} /debootstrap/debootstrap --second-stage
```

After this success another question pops up: How did Linux inject QEMU in between my aarch64 binaries and my amd64
host system? This time the magic is called [binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc). Our aarch64
executable files start with a certain "magic":

``` bash
ml@amd64-u1804:~$ hexdump -C -n 20 ${debootstrap_dir}/bin/ls
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  03 00 b7 00                                       |....|
00000014
```

The qemu-user-static Debian package has registered a handler (/usr/bin/qemu-aarch64-static) for this "magic":

``` bash
ml@amd64-u1804:~$ cat /proc/sys/fs/binfmt_misc/qemu-aarch64 
enabled
interpreter /usr/bin/qemu-aarch64-static
flags: OC
offset 0
magic 7f454c460201010000000000000000000200b700
mask ffffffffffffff00fffffffffffffffffeffffff
```

And therefore we automatically run aarch64 binaries using the QEMU emulator:

``` bash
ml@amd64-u1804:~$ sudo chroot ${debootstrap_dir} sleep 20 &
ml@amd64-u1804:~$ ps aux | grep "sleep 20"
...
root     15654  0.1  0.3  57068  7100 pts/0    Sl   10:19   0:00 /usr/bin/qemu-aarch64-static /bin/sleep 20
...
```

Fast-forward 4 years - the things have slightly changed:

# The Present - Ubuntu 22.04

Let's repeat the same procedure on Ubuntu 22.04:

``` bash
ml@amd64-u2204:~$ sudo apt-get install debootstrap qemu-user-static
ml@amd64-u2204:~$ debootstrap_dir=buster-arm64
ml@amd64-u2204:~$ sudo debootstrap --arch arm64 --foreign buster ${debootstrap_dir} http://deb.debian.org/debian
ml@amd64-u2204:~$ sudo chroot ${debootstrap_dir} /debootstrap/debootstrap --second-stage
```

Success! What the heck? We did not copy QEMU into the minimal root file system yet. Where is QEMU hiding? How did the
binaries in the arm64 root file system find it?

After some googling I found out that the binfmt_misc registration makes the difference:

``` bash
ml@amd64-u2204:~$ cat /proc/sys/fs/binfmt_misc/qemu-aarch64 
enabled
interpreter /usr/libexec/qemu-binfmt/aarch64-binfmt-P
flags: POCF
offset 0
magic 7f454c460201010000000000000000000200b700
mask ffffffffffffff00fffffffffffffffffeffffff
ml@amd64-u2204:~$ ls -al /usr/libexec/qemu-binfmt/aarch64-binfmt-P
lrwxrwxrwx 1 root root 29 Jul  5 16:47 /usr/libexec/qemu-binfmt/aarch64-binfmt-P -> ../../bin/qemu-aarch64-static
```

The interpreter got registered with the "F" [flag](https://docs.kernel.org/admin-guide/binfmt-misc.html):

> F - fix binary
> > The usual behaviour of binfmt_misc is to spawn the binary lazily when the misc format file is invoked. However,
> > this doesn't work very well in the face of mount namespaces and changeroots, so the F mode opens the binary as soon
> > as the emulation is installed and uses the opened image to spawn the emulator, meaning it is always available once
> > installed, regardless of how the environment changes.

Let's double check this behavior:

``` bash
ml@amd64-u2204:~$ sudo chroot ${debootstrap_dir} sleep 20 &
ml@amd64-u2204:~$ ps aux | grep "sleep 20"
...
root       43234  0.2  0.0 222200  7868 pts/4    Sl   10:52   0:00 /usr/libexec/qemu-binfmt/aarch64-binfmt-P /usr/bin/sleep sleep 20
...
```

Indeed - it works as designed!

This leaves us with another problem - edi can no longer inject a newer version of QEMU into the chroot or the container.

What can we do about this?

We can just overrule the binfmt_misc entries that the Debian package has done with a
[Docker container](https://github.com/multiarch/qemu-user-static) that registers a newer interpreter:

``` bash
ml@amd64-u2204:~$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
ml@amd64-u2204:~$ cat /proc/sys/fs/binfmt_misc/qemu-aarch64
enabled
interpreter /usr/bin/qemu-aarch64-static
flags: F
offset 0
magic 7f454c460201010000000000000000000200b700
mask ffffffffffffff00fffffffffffffffffeffffff
```

Let's double check that the new interpreter gets used:

``` bash
ml@amd64-u2204:~$ sudo chroot ${debootstrap_dir} sleep 20 &
ml@amd64-u2204:~$ ps aux | grep "sleep 20"
...
root       43858  0.1  0.0 226992  8696 pts/4    Sl   10:59   0:00 /usr/bin/qemu-aarch64-static /usr/bin/sleep 20
...
```

Perfect!

To revert to the previous interpreter it is sufficient to restart a systemd service:

``` bash
ml@amd64-u2204:~$ sudo systemctl restart binfmt-support
```

# Conclusion

As a result of this investigation I can remove a lot of source code from the edi project that was dealing with
downloading and injecting a new version of QEMU into the chroot respectively the container. This code has become
useless as the injected QEMU will no longer be considered. To still benefit from a new QEMU emulator edi users can
inject it using the container approach described above or install a backported qemu-user-static package or upgrade
the host OS.