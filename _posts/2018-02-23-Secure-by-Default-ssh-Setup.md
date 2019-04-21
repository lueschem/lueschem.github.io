---
author: matthias_luescher
author_profile: true
description: "A secure by default configuration is usually not the most user friendly setup. In this blog post I will show how edi does setup ssh with above average security settings and still remains user friendly."
comments: true
title: Secure By Default ssh Setup
---

Nowadays - in the funny days of IoT - the majority of embedded devices are reachable
over the network. This was not the case some years ago and therefore many embedded
developers are still dealing with static passwords to log in e.g. over ssh. This
was also the case with the initial [edi-pi](https://github.com/lueschem/edi-pi)
setup. To put it short - I felt very concerned about it.

Here is how I changed it leveraging new [edi](https://www.get-edi.io) features:

## Precondition

If you want to move away from static passwords it is an obvious choice to make use of
ssh keys. If you are a frequent GitHub user or server administrator it is very
likely that you already have a ssh key pair. You can check this as follows:


``` bash
$ ls -al ~/.ssh
```

Valid public keys are typically named `id_rsa.pub`, `id_dsa.pub`, `id_ecdsa.pub`
or `id_ed25519.pub`.

If you do not find a key pair it is a good idea to generate one now: 

``` bash
$ ssh-keygen -t rsa -b 4096 -C "you@example.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/YOU/.ssh/id_rsa):
Created directory '/home/YOU/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

A key pair consists of a public key (e.g. `id_rsa.pub`) and a matching private key
(e.g. `id_rsa`). Your duty is to make sure that nobody but you has access to the
private key. Therefore further protecting it with a passphrase is a good idea.
If you do not want to re-type the password every time you use the key, a `ssh-agent`
will do the trick for you.

## Creating the Container

Given you have installed edi according to
[this instructions](https://docs.get-edi.io/en/latest/getting_started.html)
(version >= 0.11.0) you are now ready to go.

First we build a brand new LXD container from scratch:

``` bash
$ mkdir ~/my-edi-ssh-demo && cd ~/my-edi-ssh-demo
$ edi config init my-edi-ssh-demo debian-stretch-amd64
$ sudo edi -v lxc configure my-edi-ssh-demo my-edi-ssh-demo-develop.yml
```

Since the container gets built from scratch you have plenty of time to take a
look at the output. If you have done the ssh key setup according to the instructions
above you should find something like:

```
edi_current_user_ssh_pub_keys:
- /home/YOU/.ssh/id_rsa.pub
```

`edi` has identified your public key and will automatically add it to the
`authorized_keys` files of your container users.

Furthermore `edi` will deactivate password based ssh login to the container by
modifying `/etc/ssh/sshd_config` and adding the line `PasswordAuthentication no`.

## Accessing the Container

Now we have to figure out the IPv4 address of the container:

``` bash
$ lxc list my-edi-ssh-demo
```

And now we can directly access it:

``` bash
$ ssh IP_OF_CONTAINER
The authenticity of host 'IP_OF_CONTAINER (IP_OF_CONTAINER)' can't be established.
ECDSA key fingerprint is SHA256:XYZ.
Are you sure you want to continue connecting (yes/no)? yes
```

Now you should have entered the container without the need to type in the password
of the container user (which is _ChangeMe!_).

You can leave the container again:

``` bash
$ exit
```

If you get the message `Permission denied (publickey).` something is wrong with
your ssh key setup. You might want to check again if `edi` was able to identify
your public key:

``` bash
$ edi lxc configure --dictionary my-edi-ssh-demo my-edi-ssh-demo-develop.yml 
...
edi_current_user_ssh_pub_keys:
- /home/YOU/.ssh/id_rsa.pub
...
```

Apart from the personal user in the container you can also access the container
by using the default container user (named _edi_ in this example or _pi_ in the
edi-pi setup):

```
$ ssh edi@IP_OF_CONTAINER
```

and leave it again:

```
$ exit
```

## Involving your Team

At the moment, you are the only one that is able to ssh into your container. For
a container setup this might be ok but as soon as you are using `edi` to generate
target system images (see e.g.
[this blog post](/A-new-Approach-to-Operating-System-Image-Generation/)) you might
need to authorize your coworkers to access a certain device.

Luckily this is easy! Just collect the public keys (they must have the
extension .pub) of the people you would like to authorize and drop them into
the newly created ssh public key folder:

``` bash
$ mkdir -p ~/my-edi-ssh-demo/ssh_pub_keys
$ cp *.pub ~/my-edi-ssh-demo/ssh_pub_keys/
```

Now you can apply those keys to the existing container:

``` bash
$ cd ~/my-edi-ssh-demo
$ edi -v lxc configure my-edi-ssh-demo my-edi-ssh-demo-develop.yml
```

Congratulations: Your coworkers are now also authorized to access the container
by using the default user:

``` bash
$ ssh edi@IP_OF_CONTAINER
```

## Conclusion

At the age of IoT static passwords that can be used for remote access are a no-go.
We should already avoid them during development to make 100% sure that we never
ship products that can be accessed by a static password. `edi` will help you
to make the usage of ssh keys even more convenient.

If your setup gets more complex, be aware that ssh also supports certificate
authorities since some time. Please read
[this blog post](https://framkant.org/2016/10/setting-up-a-ssh-certificate-authority-ca/)
to get started.
