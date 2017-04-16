---
author: matthias_luescher
author_profile: true
description: "Debian is a great choice for embedded projects. This blog post presents three approaches for getting your source code compiled for the target system."
comments: true
---

## Introduction

This case study has the goal to compile and package C/C++ based software for our [Raspbian](https://www.raspbian.org/) target system.

We have some hardware on our desk:

- A notebook with an i7-7500U CPU (7th Generation Intel® Core™ i7 Processor) - the details just matter for the benchmarks.
- A Raspberry Pi 3 Model B with an ARM Cortex-A53 CPU.
- Some cables.
- A monitor and a keyboard.
- A Sandisk Extreme Pro microSDXC memory card.

... and some software is installed as follows:

- Ubuntu 16.04 on the Dell ultrabook.
- Raspbian Jessie on the Raspberry Pi 3.

The explanations given below make use of the tool _edi_. If you want to reproduce 
the results on your own setup you can install _edi_ according to [this instructions](http://docs.get-edi.io/en/latest/getting_started.html).

It will not come as a surprise that the given goal can be reached with several approaches. Three of them will be shown here and I will try to
summarize the advantages and disadvantages of each approach.

To have some countable results I will do the comparison based on the following benchmark:


## The Benchmark 

We want to recompile openssl for our Raspbian based target system. The produced artifacts shall be packaged as Debian packages.

To get an initial impression, let us do a native build for Debian jessie.

1. On our notebook, we setup a Debian jessie based lxc container named _debian-jessie-amd64_:

   ``` bash
   mkdir debian-amd64
   cd debian-amd64
   edi config init debian-jessie-amd64 debian-jessie-amd64
   sudo edi -v lxc configure debian-jessie-amd64 debian-jessie-amd64-develop.yml
   ```



