--- 
layout: post
title: Marvell 88E8056 and vSphere 4.1
date: 2011-3-28
comments: false
categories: General VMware Geek
link: false
---

With VMware's latest update to vSphere they've started to include their own
copy of the sky2 driver and some support for various Marvell-Yukon NICs. 
However, out of the box the onboard NIC on my Asus P5K-Deluxe - an 88E8056 -
doesn't appear to be recognized.  At first glance it looked  to be as
straightforward as creating an updated oem.tgz with an amended
/etc/vmware/simple.map containing the relevant hardware ID but alas, that
doesn't work.  Loading VMware's sky2.o module with vmkload_mod just throws an
error regarding unresolved symbols, which suggests that the scope of supported
hardware by the driver isn't quite as broad as you'd hope.

Some digging around revealed the steps necessary to create this [homebrewed
version](http://www.kernelcrash.com/blog/using-a-marvell-lan-card-with-esxi-4/2009/08/22/).
The author provides a link to a precompiled version, but this doesn't (yet)
have the necessary ID in place in order to pick up my 88E8056.   One quick
amendment to the aforementioned simple.map and we're in business though, and
I'm happy to report that this particular NIC now works quite happily under 4.1.
To save anyone else the effort and headscratching here's a copy to drop in
place on your installation: [oem.tgz](http://dischord.org/misc/dump/oem.tgz)
