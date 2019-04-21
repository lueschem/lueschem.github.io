---
author: matthias_luescher
author_profile: true
description: "After almost three years of development edi 1.0.0 has arrived!"
comments: true
title: "Announcing edi 1.0.0"
---

Development of embedded systems has become more complex over time and nowadays groups of developers interact
with a lot of [infrastructure](/Infrastructure-at-Your-Fingertips/) to get the work done.
The goal of `edi` is to facilitate the setup of this
infrastructure on the development hosts, on the build servers, on the test infrastructure or wherever else
it is needed.

As a key principle `edi` leverages [container technology](https://linuxcontainers.org/)
to build a digital twin of the target hardware
on the development host. This boosts productivity and enables simultaneous engineering.

Now, after almost three years of spare time development effort I decided to bump the `edi` version to 1.0.0!

Here are some highlights of the things you can do with `edi`:

- Build a [cross development LXD container for Raspbian](/Cross-Compiling-for-Raspbian/), Debian or Ubuntu.
- Generate pure [Debian images for the Raspberry Pi](/A-new-Approach-to-Operating-System-Image-Generation/)
or other hardware.
- Run and debug [GUI applications within your LXD development container](/Running-GUI-Applications-Within-LXD-Container/).

For more insights please visit my [blog posts](/blog/) or the
[comprehensive documentation](http://docs.get-edi.io/en/latest/).

As of today, `edi` 1.0.0 is available as a Debian package for the following Ubuntu and Debian releases:

- Ubuntu 16.04
- Ubuntu 18.04
- Ubuntu 19.04
- Debian stretch

For installation instructions, please [start here](http://docs.get-edi.io/en/latest/getting_started.html).
