---
author: matthias_luescher
author_profile: true
description: "Thanks to improvements done on the building blocks used by edi there is a 25% speedup on Ubuntu Focal!"
comments: true
title: "25% Speedup on Ubuntu Focal"
---

[Ubuntu 20.04 (Focal Fossa)](https://releases.ubuntu.com/20.04/) is another great Ubuntu LTS release!

Even better - it brings a massive speedup for [edi](https://www.get-edi.io) users: As an example the comprehensive
test suite took 25% less time to finish: 

![Ubuntu Focal](/assets/images/blog/ubuntu_focal.png){:class="img-responsive"}

Many thanks go to Thomas Lange for
[improving the debootstrap code base](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=871835).

Please note that Ubuntu switched to nftables and therefore you might need to
[update your configuration](https://docs.get-edi.io/en/latest/upgrade_notes.html#distributions-using-nftables-instead-of-iptables)
in case your legacy or emulated container is not able to obtain an IPv4 address.
