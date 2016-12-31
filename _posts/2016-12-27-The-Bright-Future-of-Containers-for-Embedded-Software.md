---
author: matthias_luescher
author_profile: true
comments: true
---

Some three years ago I asked myself:  
What is the most efficient way to develop software for embedded devices?

Of course, there is no general answer to this question since different projects have completely
different requirements and constraints. On the projects that I have been working, I always faced
rather complex requirements on one hand but on the other hand the hardware constraints were quite
relaxed (plenty of CPU power, RAM and storage). For an engineer, this is a great starting
position since you are challenged to
fulfill complex requirements but you can apply creative strategies to overcome the roadblocks.  
This was the environment where I have discovered that Linux containers are a great match for embedded
software development.

But let us first start with the probably most popular answer to the above question:

## Cross Development Workflow

A very popular choice for embedded projects is [yocto](https://www.yoctoproject.org/). As soon as you
are developing an application using yocto your edit-compile-run cycle basically looks like this: 

![cross development workflow](/assets/images/blog/cross-compilation-cycle.png){:class="img-responsive"}

On your development host you are using an [eclipse based IDE](http://www.eclipse.org/) to edit your
source code. A cross compiler then translates your source code into binary code for the target platform
(please note that you can also emulate your target platform using [QEMU](http://wiki.qemu.org)). Then
you transfer the binaries to your (emulated) target platform and run them there.

This is a bit more complex than the workflow on popular Linux distributions
(e.g. [Ubuntu](https://www.ubuntu.com/)):

## "Self Contained" Workflow

When developing for a Linux distribution, the development host is also your target system:

![self contained workflow](/assets/images/blog/self-contained-cycle.png){:class="img-responsive"}

This is straightforward but it comes with a great drawback: As a software developer you tend to
upgrade your development host to the latest and greatest release of your favorite Linux
distribution. Unfortunately the customers that are using your embedded systems might be more conservative
about upgrading their systems. Furthermore your favourite Linux distribution might not be the best choice
for embedded devices.

To put it short: The "self contained" workflow is convincingly simple but it can not be directly applied
to embedded software development.

This is where Linux containers come to the rescue:

## Linux Container Workflow

Let us imagine that you have chosen Ubuntu for your development host and that you are planning to operate
your target system with a minimal [Debian](https://www.debian.org/) installation.

The best way to develop software for a Debian system is to use a Debian system. Therefore we equip our
development host with a container that runs the same Debian version we are also planning to ship with
our embedded solution:

![container workflow](/assets/images/blog/container-cycle.png){:class="img-responsive"}

If you are doing a clever container setup, it is very easy to share a source code folder between your
development host and your development container. By using this setup, you can either edit your source code
with your favourite IDE that is running on the development host or you can use an IDE that is running in
the development container (e.g. via X forwarding over ssh). The rest oft the cycle looks very similar to
the "self contained" workflow - the only thing that changes is that you are doing it within a container.

## Advantages of the Container Workflow

Today I am still very happy that I have started to bet on Linux containers some three years ago. Over the years
the advantages of containers by far outweighed the initial effort to get to know this new technology.
I see at least benefits in the following areas:

### Development

- Thanks to containers, I can easily run multiple different OS instances on my development host. This is especially helpful
if I do some feature development on the latest version of a container (e.g. Debian stretch based) and some bug fixing on a 
stable release that is based on an older version of a container (e.g. Debian jessie based).
- In contrast to a chroot, I can safely run a full init system such as [systemd](https://en.wikipedia.org/wiki/Systemd)
within my container (given that I choose a technology such as [LXC/LXD](https://linuxcontainers.org) - Docker and
systemd are currently not friends).
- The provisioning of a container is faster and more reliable than the bare metal provisioning of a target hardware.
- Given that the kernel of the development host is [PREEMPT_RT](https://wiki.linuxfoundation.org/realtime/start)
patched, I can also do real time in the development container.
- With a proper container setup I can make sure, that the container behaviour is very similar to the behaviour 
of the target system. For example thanks to the own networking namespace of the container it is possible to already assign
the network interface names of the future target system and even the iptables setup can be tested out.
- Containers enable me to do most of the development even without the availability of the target hardware. This
enables simultaneous engineering and reduces time to market.

### Testing

- Thanks to the fast and reliable provisioning process of a container, I can easily spin up a new container
for each testing session. This is a huge benefit for the reproducibility of the test results.
- Since the containers are lightweight, I can run multiple test sessions at the same time on the same development
host.
- Tests run faster within the container on the development host than on the (low powered) target hardware.

### Deployment

At first, I was only using containers on my development host and I never used them to ship software on
target systems. Since containers are lightweight enough, I am sure that we will see them more and more on future
embedded products:

Containers will allow us to decouple the application software stack from the Linux package of the hardware vendor:
Most of the hardware vendors sell their (ARM) boards with a yocto based board support package that is derived from
some hopefully recent yocto release. My suggestion is,
that hardware vendors concentrate on delivering a minimal board support package for their hardware which gets
regularly updated over an industry friendly time period. To make the life of the hardware vendors easier, 
they are allowed to do "rolling releases" based on the latest yocto releases. 
The six month ["rolling" release cycle of yocto](https://wiki.yoctoproject.org/wiki/Releases) is often too fast for
developing stable applications for embedded devices.
Application development might be more efficient in a Debian container that is 
[supported for about five years](https://wiki.debian.org/LTS).  
To summarize the approach: the yocto powered base system gives the hardware vendor the ability to quickly support
new hardware. At the same time the application developer enjoys a stable container based environment with long
term support.
 
On very complex deployments with multiple involved companies it might even make sense to deploy multiple
containers to a target system to get the best possible separation of concerns.

## Conclusion

Linux containers are a perfect match for many embedded products and their development. The technology is still quite
young but I am convinced that it will become commonplace over the next five years - even in the rather
conservative area of embedded software development.

To answer the initial question:  
Depending on your requirements and constraints, Linux containers might help to speed up your development process for
embedded systems! It is a good time to get familiar with this powerful technology.