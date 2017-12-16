---
author: matthias_luescher
author_profile: true
description: "This blog post outlines a (partially) new approach to OS image creation. Known tools and technologies such as debootstrap, Ansible, LXD and yaml are being used to conveniently build OS images from ground up."
comments: true
---

The last few weeks `edi` gained an important new feature: Given you have
installed `edi` according to
[this instructions](http://docs.get-edi.io/en/latest/getting_started.html)
and cloned the `edi-pi` repository from GitHub:

``` bash
git clone https://github.com/lueschem/edi-pi.git
cd edi-pi
```

and installed a few extra tools:

``` bash
sudo apt install e2fsprogs dosfstools bmap-tools
```

You are now ready to generate a minimal pure Debian stretch arm64
image for the Raspberry Pi 3 using a single command:

| Important Note |
| --- |
| OS image generation operations require superuser privileges and therefore you can easily break your host operating system. Please ensure that you have a backup copy of your data. |

``` bash
sudo edi -v image create pi3-stretch-arm64.yml
```

The resulting image can be copied to a SD card (here /dev/mmcblk0)
using the following command:

| Important Note |
| --- |
| Everything on the SD card will be erased! |

``` bash
sudo bmaptool copy artifacts/pi3-stretch-arm64.img /dev/mmcblk0
```

If the command fails, unmount the flash card and repeat the above command.

Once you have booted the Raspberry Pi 3 using this SD card you can
access it using ssh (password is _raspberry_):

``` bash
ssh pi@IP_ADDRESS
```

or via local login using keyboard and monitor.

Now we have to take a look behind the scenes to better understand how
`edi` does the OS image generation:

## The Command Pipeline

OS image generation always involves time consuming steps. An important
design goal of `edi` was to separate the steps in order to make sure that if a
step fails, that the artifacts from the preceding steps remain valid.
This definitively helps to speed up the development of new OS images
because the whole image can be developed step by step. While doing the later
steps `edi` will directly reuse the artifacts of the previous steps without
consuming time to re-generate them.

The resulting "command pipeline" looks like this:

![command pipeline](/assets/images/blog/command_pipeline.png){:class="img-responsive"}

`edi` does not even try to be clever about (intermediate) artifacts:
If the artifact is present it will take it, otherwise it will try to
generate it. If you want to force the re-creation of selected artifacts,
you can tell edi to recursively delete artifacts from the N preceding steps:

``` bash
sudo edi image create --recursive-clean 5 pi3-stretch-arm64.yml
```

This command (here N=5) will remove most of the artifacts (up to and including the
container) but e.g. the bootstrap artifacts will remain untouched.

A re-creation of the OS image will be a lot faster since the bootstrapping
gets skipped thanks to already existing artifacts:

``` bash
sudo edi -v image create pi3-stretch-arm64.yml
```

## The Toolbox

When creating OS images `edi` does not build upon a pre-built iso image but
it starts from scratch using `debootstrap`. This has a beneficial effect
especially for embedded devices since you can control all the installed
packages right from the start and you do not have to remove any unwanted
packages afterwards.
Instead of customizing the minimal root file system within a chroot `edi`
makes use of an [LXD](https://linuxcontainers.org/) container.
This has a positive impact during the
engineering phase since a lot of things (including service startup and
network setup) can already be fine tuned within the container.
The customization of the image happens within the LXD container by
means of [Ansible](https://www.ansible.com/).
When all the customization has been successfully completed, the LXD
container gets exported and some additional custom scripting converts
the root file system into the desired image format.

## The Vital Side Products

For embedded cross development it is desirable to make use of a cross
compiler to reduce compile time. Thanks to the container based
setup a single command is sufficient to generate a matching cross
toolchain:

``` bash
sudo edi -v lxc configure edi-pi-cross-dev pi3-stretch-arm64-cross-dev.yml
```

Furthermore and emulated arm64 container can be built with the following
command:

``` bash
sudo edi -v lxc configure edi-pi-dev pi3-stretch-arm64-dev.yml
```

The following picture shows an overview of the artifacts we have seen
so far:

![container and image artifacts](/assets/images/blog/edi_pi_artifacts.png){:class="img-responsive"}






