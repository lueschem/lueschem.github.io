---
author: matthias_luescher
author_profile: true
description: "Debian is very popular in the cloud and on embedded devices. It is pretty easy to build tailored OS images but some things can go wrong. Here is a list that helps you to bypass 11 possible traps."
comments: true
---

The target audience of this blog post are either people that are using tailored Debian images or
people that create or plan to create tailored Debian images. The checklist below shall be useful for
all of them to verify the correctness of a given or resulting image.

![traps](/assets/images/blog/image_traps.png){:class="img-responsive"}

Here are the eleven traps that I have encountered during my past 5+ years of experience with building
tailored Debian images:

## 1. ssh Host Keys

During the installation of the `openssh-server` package the post installation script will automatically
generate ssh host keys. This is very convenient if you install `openssh-server` on an existing machine.
However, if you want to create a distributable image the ssh host keys must be unique for each
installation because otherwise multiple systems would identify themselves with the same key. Therefore
you have to do two things with your distributable image:

- Delete the generated ssh host keys (`/etc/ssh/ssh_host_*_key*`) before you distribute your image.
- Provide a service that re-generates the ssh host keys on the first boot of the image.

The key generation service can look like this:

``` bash
[Unit]
Description=Generate host keys for openssh-server.
Before=ssh.service

[Service]
Type=oneshot
ExecStart=/usr/bin/ssh-keygen -A -v

[Install]
RequiredBy=multi-user.target
```

If you are interested in more details, please take a look at the 
[base_system_cleanup role](https://github.com/lueschem/edi/tree/master/edi/plugins/playbooks/debian/base_system_cleanup/roles/openssh_server_keys)
of `edi`.

## 2. Installation of Software Packages

Sometimes you might want to install a software and you can not find that software packaged as a Debian
package. A misguided approach would be to just copy and paste that software into the image. However, you
will deeply regret having taken that shortcut as soon as it comes to a software update. The correct
approach is to package this software as a proper Debian package. There is a certain
learning curve to get used to it
but it is well worth the journey. To get started with Debian packages you can take a look at the
[Debian manuals](https://www.debian.org/doc/manuals/debmake-doc/ch04.en.html). And yes - just to make
sure that you bypass another trap - I would like to mention this
[blog post](https://raphaelhertzog.com/2010/12/17/do-not-build-a-debian-package-with-dpkg-b/).

## 3. Choosing a Mirror

Some time ago it was commonplace to choose a country specific Debian mirror when creating an image.
This is kind of problematic if you assemble an image in Switzerland and then it is being used in
India. The package downloads will then be slow because they make a detour over Switzerland.
Luckily there is a solution for this thanks to the sponsoring of
[Amazon CloudFront](https://aws.amazon.com/de/cloudfront) and [Fastly](https://www.fastly.com).
Nowadays you can just take the
[content delivery network](https://en.wikipedia.org/wiki/Content_delivery_network) address and a
sources.list file might then look like this:

```
deb http://deb.debian.org/debian stretch main contrib non-free
deb http://deb.debian.org/debian stretch-updates main contrib non-free
deb http://deb.debian.org/debian stretch-backports main contrib non-free
deb http://security.debian.org/ stretch/updates main contrib non-free
```

The content delivery network will automatically choose the right mirror for you.

## 4. Trustworthy Software Sources

Debian typically downloads packages over `http`. This would be unsafe if there would not be a check
against a GPG signature. This signature check must already happen during the initial bootstrapping
process, otherwise the image might already be compromised right from the start. The `debootstrap` tool
can be forced to check the GPG signatures by using the `--force-check-gpg` option. Obviously you have
to provide the appropriate verifications keys to `debootstrap` which can be downloaded e.g. from
`https://ftp-master.debian.org/keys`. Please note that you should download keys only from a
trustworthy source (e.g. not over `http`).

## 5. Reproducibility

The assembly of a complex image involves several steps. Make sure that you automate all those steps
in order to generate images with reproducible content. Tools like [edi](https://www.get-edi.io),
[elbe](https://elbe-rfs.org/)
or [packer](https://www.packer.io/) will help you to achieve this goal.

## 6. Login Credentials

If you are distributing an OS image to an unknown audience you have to provide the login credentials
to that audience. Typically this is a user name and a static password that is the same for all images.
As a minimal safety precaution you should immediately change that password if you have created a
new instance from a given image. `edi` goes even
[a step further](/Secure-by-Default-ssh-Setup/) and disables password based login
over ssh by default. With this precaution even an unchanged default password is not exploitable over
the network.

The [Ubuntu Core](https://www.ubuntu.com/core) team does a great job in adding
your public key to the image during the download of the image!

## 7. Service Startup During Installation

If you are assembling an OS image you should make sure that you do not start services during package
installation because those services might generate some unique IDs during their first startup.
Fortunately `dpkg` has foreseen this use case and you can add a file `/usr/sbin/policy-rc.d`
(mode `755`) to your image with the following content:

```
exit 101
```

This will prevent the service startup from within the post installation scripts (please note
that post installation scripts shall not call `systemd` directly). At the end of the image assembly
you have to remove the file `/usr/sbin/policy-rc.d` again.

## 8. Machine ID

Your Linux system contains a file named `/etc/machine-id` containing a unique identifier for your
system. It is easy to guess what happens if this file is already part of the OS image: The
content of `/etc/machine-id` will not be unique anymore if you install that image on multiple
instances. This is why you have to remove `/etc/machine-id` from your OS image.
`systemd` will re-create that file on the first boot but this also triggers some special behaviour
of `systemd`:

## 9. systemd Preset Behavior

Let us assume that your OS image has a service installed that should just be available for developers.
We call that service `unsafe-dev-backdoor.service`. Since we do not want to have that service running on
production images we disable and stop it during image creation:

``` bash
sudo systemctl disable unsafe-dev-backdoor
sudo systemctl stop unsafe-dev-backdoor
```

Now you go into production with that image and guess what: All your devices will be delivered with
the `unsafe-dev-backdoor.service` enabled and running!

The removal of `/etc/machine-id` will trigger a first start behavior of `systemd`. During that process
`systemd` will preset all service to "factory defaults" and this will enable and start the
`unsafe-dev-backdoor.service`.

To change the "factory defaults" you can put a file named `10-unsafe-dev-backdoor.preset`
to `/lib/systemd/system-preset` with the following content:

```
disable unsafe-dev-backdoor.service
```

Now your delivered devices are safe again!

## 10. File System Permissions

The GNU `tar` utility comes with some built-in intelligence to map user and group _names_ to their
_numerical values_. If we use Ubuntu to build a Debian image this built-in intelligence can
get confused in some edge cases. Luckily you can disable this built-in intelligence by using `tar`
with the option `--numeric-owner`. It is a good idea to use that option in every place where
your image scripts do `tar` or `untar` operations.

## 11. Reducing Image Size but Remain Legally Compliant

Depending on your use case you do not need some content of the `/usr/share` folder and therefore
you can tell `dpkg` to skip the installation of the contents of certain folders. This will have
a measurable impact on the image size. `edi` for instance allows you to disable the installation of
documentation via `/etc/dpkg/dpkg.cfg.d/01_no_documentation`:

```
# save disk space by excluding documentation
path-exclude /usr/share/doc/*
# keep copyright files for legal reasons
path-include /usr/share/doc/*/copyright
path-exclude /usr/share/man/*
path-exclude /usr/share/groff/*
path-exclude /usr/share/info/*
path-exclude /usr/share/lintian/*
path-exclude /usr/share/linda/*
``` 

Please note the fourth line of this config file: `edi` will make sure that the copyright files
do get installed to provide a legally compliant image.

## Conclusion

I hope that the above checklist helps you to confirm that you
or your image provider have bypassed all possible traps.

The images generated using the tool [edi](https://www.get-edi.io) with the configuration
[edi-pi](https://github.com/lueschem/edi-pi) will also come with a proper setup according to
the above checklist.

Do you know additional traps that would be worth mentioning here? I will really appreciate
your input!
