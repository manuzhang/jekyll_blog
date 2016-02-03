---
comments: true
layout: post
title: Unikernel 
---

The first time I learned about Unikernel was from [What is a ‘unikernel’?](https://ma.ttias.be/what-is-a-unikernel/) when glancing through posts from Pocket Recommended on my shuttle bus the other day. It's totally out of curiosity and "Recommended" usually means a lot of people are watching it. After a quick read, I didn't get the point. Is it something [Docker](http://roadtounikernels.myriabit.com/) ? I forgot about it as soon as I got off the shuttle bus.

Then came the news that [Unikernel Systems Joins Docker](http://blog.docker.com/2016/01/unikernel/). Interesting ! Meanwhile, I was looking for materials for weekly internal sharing at work. Why didn't I share about Unikernels ? I went on to google and found a bunch of resources from [Wikipedia](https://en.wikipedia.org/wiki/Unikernel), [Xen Wiki](https://wiki.xenproject.org/wiki/Unikernels
) and [unikernel.org](http://unikernel.org/).

## Why do we need Unikernel

At the age of cloud computing, programmers Tom and Jerry rent two Linux virtual machines (VMs) from [Amazon EC2](https://aws.amazon.com/ec2) to host their web servers, and the two VMs end up on the same physical machine. The enabing technique is called virtualization.

![road_to_unikernels](https://ma.ttias.be/wp-content/uploads/2015/11/road_to_unikernels.png)

(Source: [road to unikernels](http://roadtounikernels.myriabit.com/)) 

As in the above image, hardware resources are virtualized by a [hypervisor](https://en.wikipedia.org/wiki/Hypervisor) and shared among multiple VMs. One of the most well known hypervisors, [Xen](http://www.xenproject.org/) (used by EC2, Rackspace, etc), is developed by the [University of Cambridge Computer Laboratory](https://en.wikipedia.org/wiki/University_of_Cambridge_Computer_Laboratory), **who conceived Unikernels**.

> Despite this shift from applications running on multi-user operating
systems to provisioning many instances of single-purpose
VMs, there is little actual specialisation that occurs in the image
that is deployed to the cloud. We take an extreme position on specialisation,
treating the final VM image as a single-purpose appliance
rather than a general-purpose system by stripping away functionality
at compile-time

(Source: [Unikernels: Library Operating Systems for the Cloud](http://anil.recoil.org/papers/2013-asplos-mirage.pdf) )

If Tom and Jerry only use the VMs to host their web servers, they probably don't need a general-purpose system for a desktop computer. With a single-purpose Unikernels system, Tom and Jerry could link only the system functionality required by their web servers. 

The benefits are 

* improved security with reduced deployed codes and attack surface
* small footprints and fast boot time
* whole-system optimisation across device drivers and application logic

## Now, what is a Unikernel

> Unikernels are specialised, single-address-space machine images constructed by using library operating systems.

(Source: [Unikernel](https://en.wikipedia.org/wiki/Unikernel))

Unikernel is built on library operating system (libOS).

### Library Operating System

In a library operating system,

* there is only a single address space
* device drivers are implemented as a set of libraries
* access control and isolation are enforced as policies in the application layer

Without the separation of user space and kernel space, there is no need for repeated privilege transitions to move data between them. On the other hand, the single address space makes it difficult to isolate mulitple applications running side by side on a libOS. 

The drawback has been overcame by modern hypervisor which provides virtual machines with CPU time and strongly isolated virtual devices.

Another drawback is protocol libraries (e.g. device drivers) have to been rewritten to replace that of a tradtional operating system. 

Fortunately, they have been implemented in [MirageOS](https://mirage.io/), the first prototype of a Unkernel. 

### MirageOS

![unikernel architecture](https://upload.wikimedia.org/wikipedia/commons/b/b3/Unikernel_mirage_example.png)

The traditional software stacks are reduced to application code and mirage runtime. OS functionalities are imported as libraries provided by MirageOS. Every application is compiled into its own specialised OS.

Users could target their codebase towards different platforms, Unix for test and Xen on x86 or Xen on ARM for deployement, which are run on the cloud or embedded systems. 

Besides MirageOS, there is an [expanding Unikernel ecosystem](http://unikernel.org/projects/). 

> Some systems (like Rumprun) are language-agnostic, and provide a platform for any application code based on the requests it makes of the operating system... Unikernels can run in containers, on hypervisors, and on a wide array of bare-metal hardware.


## Is Unikernel really needed

My colleagues are uncovinced. On the internal sharing, they don't think it's worthwhile to throw all the existing software stacks in the Linux ecosystem. Meanwhile, the benefits brought by Unikernel are doubtable. For example, tailored Linux systems could be also small. [docker-alpine](https://github.com/gliderlabs/docker-alpine) image is only 5 MB.

At the time of writing, another post popped up from Pocket Recommended, [Unikernels are unfit for production
](https://www.joyent.com/blog/unikernels-are-unfit-for-production). The author questions the main reaons (performance, security, footprint) to adopt a Unikernel system. He thinks the most important reason that Unikernel is unfit for production is that **Unikernels are entirely undebuggable** (please read the original article for thorough explanations).

## Summary

In this post, we get to know what Unikernel is and why it exists. It's unclear whether Unikernel will become a new revolution for cloud computing like Docker (I also list the discussions on Hacker News and Reddit in the reference below). 

## References

1. [Unikernels: Rise of the Virtual Library Operating System](http://queue.acm.org/detail.cfm?id=2566628) by Anil Madhavapeddy and David J. Scott.
2. [Unikernels at PolyConf 2015](https://speakerdeck.com/amirmc/unikernels) by Amir Chaudhry.
3. [Discussions on Hacker News](https://news.ycombinator.com/item?id=10945219)
4. [Discussions on Reddit](https://www.reddit.com/r/programming/comments/4206cv/unikernel_systems_joins_docker/)



