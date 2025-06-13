---
author: matthias_luescher
author_profile: true
description: "Nowadays there are fast computers with arm64 architecture. It is high time that these computers are used profitably for embedded development."
comments: true
title: "Speed up Embedded Development on an arm64 Host"
---

I grew up at a time when embedded development used to happen on a powerful amd64 machine. To create binaries for the
target architecture, cross-compilation was the magic tool. To run binaries of the foreign architecture on the fast
host, emulation was your friend.

This formerly fast computer is still on my desk. It's a notebook with a 7th generation Core i7 (2016). The palm rest is
already a bit brittle as it is slowly disintegrating. We can still build a Debian OS image with it:

```
lueschem@xps13:~/edi-workspace/edi-pi$ time edi -v project make pi5.yml

...

real 15m6.424s
user 14m9.305s
sys 1m52.156s
```

Not bad — within a quarter of an hour you get a complete OS image for the Raspberry Pi 5 plus a Mender update artifact
and some documentation. This results in a relaxed workflow: Start to build, take off with colleagues for a round of
table soccer, return, see you've messed up, start building again, take off with other colleagues for coffee, return...

At the end of the working day, the job may not be done, and you try to fix it with a night shift - or - you rethink your
setup.

I prefer the second approach. I've had a NanoPi R6C with a RockChip RK3588S CPU on my desk for some time now. I donated
a fast SSD for this computer and equipped it with Ubuntu 24.04 (installed without OverlayFS - you can disable
it within the installer, the rocket ice cream is experimental and the tests were conducted without it):

[![New Setup](/assets/images/blog/arm64_speedup.png){:class="img-responsive"}](/assets/images/blog/arm64_speedup.png)

Now we repeat the same OS image build:

```
lueschem@NanoPi-R6C:~/edi-workspace/edi-pi$ time edi -v project make pi5.yml

...

real 6m40.203s
user 4m8.399s
sys 2m30.311s
```

Wow - with this rather modest computer, we are already twice as fast as before.

That is why this is possible: The OS image we are building has the same architecture as the host we are running the
build on. As a result, we are no longer slowed down by emulating a foreign architecture.

But couldn't it be a little quicker? Sure - let's take a look at the stage where we customize the OS image using
Ansible playbooks.

[![Workflow v2](/assets/images/blog/workflow-v2.png){:class="img-responsive"}](/assets/images/blog/workflow-v2.png)

Ansible starts a Python interpreter within the target system for every task.
The [Mitogen for Ansible](https://mitogen.networkgenomics.com/ansible_detailed.html) connection plugin can be used
to reduce this overhead.

With Mitogen enabled — we reproduce the OS image build:

```
lueschem@NanoPi-R6C:~/edi-workspace/edi-pi$ time edi -v project make pi5.yml

...

real 3m11.195s
user 2m14.729s
sys 1m6.023s
```

Cool - another acceleration by a factor of two!

> **Warning:**
>
> Some Ansible tasks might fail when Mitogen is being used. Those tasks can usually be fixed by adding the following
> variable to the task:
> 
> ```
>   vars:
>     mitogen_task_isolation: fork
> ```

Now we get ambitious and try to push the time under 3 minutes.
Building OS images for Debian involves downloading a lot of packages. We will now set up a local cache for these Debian
packages using `apt-cacher-ng`. The cache is being used during both — the bootstrap phase and the OS image customization
phase. A first build is done to fill the cache.

We are eagerly awaiting the result of the second build:

```
lueschem@NanoPi-R6C:~/edi-workspace/edi-pi$ time edi -v project make pi5.yml

...

real 3m1.459s
user 2m11.037s
sys 1m3.833s
```

Oh, we missed our target. The Debian infrastructure plus my internet connection seem too fast to make the cache shine.

> **Remark:**
>
> Caching is currently not a built-in feature of `edi`. As soon as I find a way to make it easy to use with reasonable
> additional effort, I'll consider adding it.

We're not giving up — faster is better. `edi` has a builtin feature that you do not have to redo everything from
scratch. During a typical development workflow the initially bootstrapped root file system does not change frequently.
So let's keep it (`--recursive-clean 2`) when we do the next image build:

```
lueschem@NanoPi-R6C:~/edi-workspace/edi-pi$ edi project make --recursive-clean 2 pi5.yml

...

lueschem@NanoPi-R6C:~/edi-workspace/edi-pi$ time edi -v project make pi5.yml

...

real	2m37.693s
user	2m3.823s
sys	1m0.772s
```

Fantastic — this is quite a bit faster than the 15 minutes we had initially!

But sometimes I mess up a task when customizing the Ansible playbook and want to jump straight to it without running
the whole playbook again.

Don't worry — this is also possible (using `--start-at-task`):

```
edi -v --start-at-task "mender : Write mender main configuration." project make pi5.yml
```

# Conclusion

The above considerations show how efficiency at work can be significantly increased and coffee consumption reduced.
This frees up time to play table soccer or indulge in another hobby in the evening.

To increase efficiency even further, I'm thinking of replacing my aging notebook with a MacBook (I would run Ubuntu
in a VM) or a Dell XPS 13 with a Snapdragon X Elite CPU (as soon as the Linux support is usable).

Bonus — I no longer need a cross-compiler to compile the packages for the target system. This also saves a lot of
headaches with complex packages. A suitable development container including compiler can be easily created with `edi`.
