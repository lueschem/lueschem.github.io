---
author: matthias_luescher
author_profile: true
description: "This blog post presents a down-to-earth approach to tackling the challenges of modern embedded systems by applying GitOps strategies."
comments: true
title: "Embedded Meets GitOps"
---

This blog post presents a down-to-earth approach to tackling the challenges of modern embedded systems that are ...

- ... connected to the internet and shall stay connected (Dispatching a technician to Ouagadougou, Pozo Hondo or
Hinterfultigen to reboot an IoT device because a developer has messed up a firewall setting might be a
little expensive.).
- ... extremely diverse and every device might require a unique configuration.
- ... highly complex and come with a huge software stack that will soon cry for bug fixes and security updates.
- ... maintained by a large group of developers with a varying level of professional experience.

The approach presented here combines ...

- ... software dinosaurs like [Debian](https://www.debian.org/) with
- ... modern tools like [Ansible](https://www.ansible.com/), [git](https://git-scm.com/)
and [Mender](https://mender.io/) by applying
- ... best practices like [GitOps](https://www.weave.works/blog/gitops-operations-by-pull-request) and
[infrastructure as code (IaC)](https://en.wikipedia.org/wiki/Infrastructure_as_code).

The outcome is a ...

- ... highly scalable, robust, traceable system that
- ... is easy to operate with a reduced number of tools and a small server infrastructure in an
- ... uncommon way that might require a shift in mindset.

Build
-----

As a first step we build our embedded system based on Debian. The developers will write source code and push it to a
git server. The image below only shows the master branches of the git repositories under the assumption that the
developers are using something like [git flow](https://nvie.com/posts/a-successful-git-branching-model/) to collaborate
and merge new features to the master branches:

![repository handling](/assets/images/blog/repository-handling.png){:class="img-responsive"}

After every commit the CI toolchain will turn the source code into a binary Debian package and upload it to a
development stage apt repository. The brave developer will of course always update his devices against this rolling
release development repository and fix regressions that are eventually popping up immediately.

At certain intervals we do a snapshot of the rolling release repository. The snapshot shall be dispatched to a limited
number of carefully selected [canary devices](https://en.wiktionary.org/wiki/canary_in_a_coal_mine) to make sure that
the new release is able to cope with the sheer amount of different configurations we have out in the field. Some
regressions are still to be expected and selected fixes will be picked from the rolling release and added to
the snapshot.

After we have made sure that no canary gets killed anymore we are ready to promote the unmodified snapshot repository
to the production stage.

By using [edi](https://www.get-edi.io) we will build images for each stage (development, canary and production). For
each stage at least two types of images will get generated:

- The complete device image that can be flashed onto the storage of the device.
- An update artifact that can be dispatched to devices over the air.

Provision
---------

To streamline the supply chain we will provision the embedded devices with a generic operating system image with a
well thought out partition layout:

![partition layout](/assets/images/blog/partition_layout.png){:class="img-responsive"}

The disk layout already foresees a redundant root file system partition that allows us to do major updates in a robust 
and reproducible way over the air. In our example we use Mender for this robust over the air update. The details
are described in a [previous blog post](/Updating-a-Debian-Based-IoT-Fleet/).

Configure
---------

After the provisioning we will have a device that is able to cope with common concerns like over the air updates and
monitoring. However, the specific use case still needs to get configured. The cloud folks already came up with good
tools like [cloud-init](https://cloud-init.io/) to customize cloud instances. For embedded devices there is still a way
to go but brilliant engineers came up with
[highly scalable systems](https://www.ansible.com/hubfs//AnsibleFest%20ATL%20Slide%20Decks/AnsibleFest%202019%20-%20Scaling%20Ansible%20for%20IoT%20Deployments.pdf)
that are a perfect match for our requirements.

First we create a git repository that describes our fleet of embedded devices using inventories and Ansible playbooks:

![infrastructure as code](/assets/images/blog/infrastructure_as_code.png){:class="img-responsive"}

The different groups of devices get attached to different git branches.

The development devices will automatically pull the Ansible playbooks from the master branch using
[ansible-pull](https://docs.ansible.com/ansible/latest/cli/ansible-pull.html). While running the playbook on the
development device apt gets configured to fetch the packages from the master repository.

At some point the developers are confident that the master branch works very stable and they issue a pull request from
master to canary. Depending on the level of automation the pull request either gets approved manually by different
stakeholders or by a fully automated process or by a manual/automated hybrid solution. After the merge to the canary
branch the canary devices now will start to pull updates from a newer snapshot.

After the new snapshot and its corresponding git IaC repository branch have proven to do no harm to the canary devices
it is now time to promote the whole changes to the production system. Please note that neither code changes on the
IaC git repository nor apt repository modifications are required to get all the changes promoted to the production
stage. The promotion of the latest software release to production is done by a simple merge of the canary git IaC
repository branch to the production branch!

Monitor
-------

As a final step we have to monitor our fleet with appropriate tools like [Icinga](https://icinga.com/) or
[Datadog](https://www.datadoghq.com/).

Such tools are well known to operations people but they are equally beneficial for developers. Especially the canary
devices require a very detailed monitoring to make absolutely sure that we do no serious harm to this lovely birds.

Also the developers might want to track their development devices and get some alerts if components are eating up RAM
due to memory leaks or services get knocked out due to a firewall change.

Conclusion
----------

All building blocks mentioned in this blog post are proven in use. Here they get combined to a robust, scalable system
that is easy to maintain.

The dramatic change is, that all over the value chain the same tools get used. An operations guy no longer uploads a
release to a shiny, dedicated device management system, selects a group of devices and dispatches the software to that
group.
Instead, he reviews a git pull request, hits the merge button and can peacefully enjoy after-work hours while the
embedded devices will get updated gradually. The careful testing on the canary devices has given enough evidence, that
the roll out will finish smoothly. If still something should go wrong, the powerful monitoring and alerting
infrastructure will make sure that the operations guy returns to the office soon. In case something went amiss badly,
there is still the possibility to re-provision the affected devices using Mender.

Also the developers will enjoy the powerful operations tools that help them to monitor the behavior of the latest
development code.

The GitOps mindset is already very popular among the [kubernetes](https://kubernetes.io/) folks but it makes a lot of
sense for the embedded people too!