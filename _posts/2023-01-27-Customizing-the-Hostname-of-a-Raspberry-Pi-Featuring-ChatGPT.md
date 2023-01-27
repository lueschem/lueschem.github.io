---
author: matthias_luescher
author_profile: true
description: "Properly changing the hostname in an automatic manner during the first boot is a non-trivial task - let's ask ChatGPT for help!"
comments: true
title: "Customizing the Hostname of a Raspberry Pi - Featuring ChatGPT"
---

It was on my TODO list since a long time: For my
[GitHub Actions runner setup](/Building-and-Testing-OS-Images-with-GitHub-Actions/) I need unique hostnames per
Raspberry Pi so that I can scale up to multiple runners. Of course - you know me by now - the setup shall be fully
automatic and the hostname shall get assigned without any human interaction during the first boot. A unique identifier
of the Raspberry Pi is its MAC address of the ethernet interface. So my fleet shall be organized like the following
picture:

![Hostname Setup](/assets/images/blog/hostname-setup.png){:class="img-responsive"}

Doing the setup for an automatic hostname assignment is not rocket science, but there are quite a few pitfalls. Let's
see how we tackle the job together with [ChatGPT](https://chat.openai.com) from [OpenAI](https://openai.com/): 

# Chat

**Me:** How can I change the hostname on a Debian system on the first boot?

---

**ChatGPT:** One way to change the hostname on a Debian system during the first boot is to use the "preseed"
configuration file. This file can be used to automate the installation process and set various system parameters,
including the hostname.

You can create a preseed file with the desired hostname and include it on the installation media or make it available
over the network. The installer will use the settings in the preseed file during the installation process.

Another way is to use the "hostnamectl" command after the system is booted up. The hostnamectl command can be used
to change the hostname of the system.

```
sudo hostnamectl set-hostname new_hostname
```

You can also edit the /etc/hostname file and change the hostname in the first line of the file.

```
sudo nano /etc/hostname
```

Then reboot your system for the changes to take effect.

_Assessment: Amazing - with a very simple question I already got a helpful answer. ChatGPT lists two possible approaches
on how to change the hostname. The response is understandable and - to be honest - ChatGPT knows the English language
better than me. Let's get a bit more specific with the requirements:_

---

**Me:** How can I automate this using a systemd service?

---

**ChatGPT:** You can create a systemd service to automatically set the hostname on boot.
First, create a new file in the `/etc/systemd/system` directory called `sethostname.service` with the following
contents:

```
[Unit]
Description=Set Hostname

[Service]
Type=oneshot
ExecStart=/bin/bash -c "echo new_hostname > /etc/hostname; hostnamectl set-hostname new_hostname"

[Install]
WantedBy=multi-user.target
```

Make sure to replace "new_hostname" with the desired hostname.

Then, you need to reload the systemd daemon to recognize the new service:

```
sudo systemctl daemon-reload
```
Enable the service to run at boot time:

```
sudo systemctl enable sethostname.service
```

You can check the status of the service with:

```
sudo systemctl status sethostname.service
```

Finally, you need to reboot the system to apply the hostname changes made by the service.

This service will run at boot time and set the hostname of your Debian system to the desired value specified in
the service file.

_Assessment: The proposed approach looks promising! ChatGPT even gives me some hints on how I can debug the setup._

---

**Me:** Sure - let me try this!

---

**ChatGPT:** Sure! I'm glad I could help. Let me know if you have any other questions or run into any issues while
setting up the service.

_Assessment: Very polite. I am glad that ChatGPT offered me some additional help. Indeed, the above setup did change
the hostname but since I deployed this new service there seems to be a glitch when using `sudo`:_

---

**Me:** Great - it worked! But now I have a small problem: Whenever I execute a sudo command, I get the following
error: "sudo: unable to resolve host newhostname: Name or service not known"

---

**ChatGPT:** This error is caused because the hostname is not properly set in the `/etc/hosts` file.
When you set the hostname with the hostnamectl command or the /etc/hostname file, the hostname is only set for
the running system, but it is not set in the `/etc/hosts` file.

To resolve this issue, you need to update the `/etc/hosts` file to include the new hostname.
You can do this by running the following command as root: 

```
echo "127.0.0.1 localhost new_hostname" >> /etc/hosts
```

Make sure to replace "new_hostname" with the hostname you want to set.

You should also include the hostname in the first line of the /etc/hosts file if it's not already there

```
127.0.0.1       localhost new_hostname
```

You can also use the `sed` command to update the /etc/hosts file.

```
sudo sed -i "s/^127.0.0.1.*/127.0.0.1\tlocalhost\t$(hostname)/" /etc/hosts
```

After updating the /etc/hosts file, you may need to log out and log back in or reboot the system for the changes
to take effect.

This should resolve the issue and you should no longer receive the "unable to resolve host" error message when
running sudo command.

_Assessment: World class - ChatGPT immediately points into the right direction. The above `sed` command shall not
blindly be applied as the Debian hosts file looks slightly different from what ChatGPT expects. After intelligently
following the above advice the `sudo` problem is gone._

---

**Me:** Perfect - this does the trick! However, I am a bit worried that the above systemd service does kick in a bit
too late.
Can we somehow make sure that the service starts early during bootup?

---

**ChatGPT:** Yes, it's possible to make sure that the `sethostname.service` starts early during the boot process by
modifying the `WantedBy` parameter in the `sethostname.service` file.

The `WantedBy` parameter in the `[Install]` section of the service file determines when the service should be
started.
By default, it is set to `multi-user.target`, which means that the service will start after the system has reached
the multi-user target state, which is relatively late in the boot process.

If you want to make sure that the service starts earlier in the boot process, you can set the `WantedBy`
parameter to `basic.target` or `sysinit.target`.

`basic.target`: This is the default target when system boots up and most of the basic services are started.

`sysinit.target`: This is the first target that is reached during boot process, and it is responsible for basic
initialization of the system.

So you can change the `WantedBy` parameter in the `sethostname.service` file to `sysinit.target`

```
[Install]
WantedBy=sysinit.target
```

Then, you need to reload the systemd daemon and enable the service again

```
sudo systemctl daemon-reload
sudo systemctl enable sethostname.service
```

Keep in mind that setting the service to start at `sysinit.target` it will be started before most of the basic
services. So it's important to make sure that the service is not dependent on any other service that may not be
started yet.

Please also note that changing the `WantedBy` parameter to `sysinit.target` may not work in all systems, and it
may have an impact on the boot process, so you should test it before deploying it in a production environment.

_Assessment: ChatGPT takes my concerns seriously and offers additional help. The ordering of systemd services
is a bit more involved than what ChatGPT thinks it to be. In fact, editing just the `[Install]` section does not do
the trick as some additional statements are needed in the `[Unit]` section too to get things done. I will give more
details further down._

_Early during boot, not all services are available yet. `hostnamectl` will fail because of the missing `udev`
service. Let's ask ChatGPT for some more help:_

---

**Me:** hostenamectl set-hostname requires udev. I guess that before sysinit.target we do not have udev. How can I
get around this issue?

---

**ChatGPT:** You are correct that the `hostnamectl set-hostname` command requires udev to be running, and udev is
typically started after the `sysinit.target`.

One way to work around this issue is to use a simple bash script that sets the hostname by directly editing
the `/etc/hostname` file, instead of using the `hostnamectl set-hostname` command.

You can create a script named `set-hostname.sh` with the following contents:

```
#!/bin/bash
echo "new_hostname" > /etc/hostname
```

Make sure to replace "new_hostname" with the desired hostname.

Then, you can modify the `sethostname.service` file to run this script on boot

```
[Unit]
Description=Set Hostname

[Service]
Type=oneshot
ExecStart=/usr/bin/bash /path/to/set-hostname.sh

[Install]
WantedBy=sysinit.target
```

You can also use the `sed` command in the script to update the /etc/hosts file.

```
#!/bin/bash
echo "new_hostname" > /etc/hostname
sed -i "s/^127.0.0.1.*/127.0.0.1\tlocalhost\t$(hostname)/" /etc/hosts
```

After that, you need to make the script executable by running the following command:

```
sudo chmod +x /path/to/set-hostname.sh
```

Then, you need to reload the systemd daemon, enable the service, and reboot the system to apply the changes

```
sudo systemctl daemon-reload
sudo systemctl enable sethostname.service
sudo reboot
```

This way you are not relying on udev to set the hostname and you can set it early on the boot process.
Please keep in mind that this method may have some limitations and you may need to test it before deploying it
in a production environment.

_Assessment: Fascinating - we get closer and closer to a good solution. I am thrilled by the experience and I would
like to write about it on my blog:_

---

**Me:** Am I allowed to post this chat on my blog post?

---

**ChatGPT:** As a language model AI, I do not have personal rights, but the contents generated by me are property
of OpenAI. You would need to check OpenAI's terms and conditions, and their API use policy. However, most likely
they would allow you to post a chat log on your blog post as long as you give proper credit and a link to OpenAI
website and mention that the answers were generated by OpenAI's GPT model.

_Assessment: I am not a lawyer but still I checked [OpenAIs](https://openai.com)
[terms of use](https://openai.com/terms/). As requested, the above answers are clearly attributed to ChatGPT.
I am under the impression that OpenAI is OK with me posting this chat._

---

**Me:** Many thanks for your help!

---

**ChatGPT:** You're welcome! I'm glad I could help. If you have any more questions or need further assistance, feel
free to ask.

# Conclusion

A great compliment to the engineers and visionary leadership behind ChatGPT! What I have seen so far is way above my
expectations. The answers provided by ChatGPT are at least as good as the answers I would get from a junior engineer -
the response time is just much faster.

The challenge I gave to ChatGPT is not trivial. I thought a bit further about it and based on the systemd-firstboot
setup I came up with the following solution for the service file:

```
[Unit]
Description=Set Hostname on First Boot
DefaultDependencies=no
Conflicts=shutdown.target
After=systemd-remount-fs.service
Before=sysinit.target first-boot-complete.target shutdown.target
Wants=first-boot-complete.target
ConditionPathIsReadWrite=/etc
ConditionFirstBoot=yes

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/edi-set-hostname

[Install]
WantedBy=sysinit.target
```

The edi-set-hostname script does in essence the following:

```
#!/bin/bash

...

OLD_HOSTNAME=$(cat /etc/hostname)
HOSTNAME_SUFFIX="$("${IDENTITY_SCRIPT}" | cut -d= -f2 | sed 's/://g')"
# take care of idempotency by not adding the suffix twice
NEW_HOSTNAME="${OLD_HOSTNAME%-${HOSTNAME_SUFFIX}}-${HOSTNAME_SUFFIX}"

# static hostname
echo "${NEW_HOSTNAME}" > /etc/hostname
# transient hostname
hostname "${NEW_HOSTNAME}"
sed -i "s/^127.0.1.1.*/127.0.1.1   ${NEW_HOSTNAME}/" /etc/hosts
```

The above service is part of the upcoming [edi-pi](https://github.com/lueschem/edi-pi)
[Debian bookworm setup](/Debian-Bookworm-on-Raspberry-Pi/)!

Later during the evening I asked ChatGPT for some advice on baking a "pot bread". This time I expressed my questions in
German and despite the language and topic change the answers remained very solid. Unfortunately it got too late to test
out the recipe.
