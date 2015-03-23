--- 
layout: post
title: ESXi, ZFS, RDM, FreeBSD, and the HP N36L MicroServer
date: 2011-6-29
comments: false
categories: General Geek VMware FreeBSD
link: false
---

Backups have started to become something of a concern now that my ageing
Netgear ReadyNAS NV+ is sporting 8TB of disk and a ~ 5.4TB filesystem.  Only a
small fraction of that is media that I really, really care about (i.e photos,
documents, content I've created) but the time taken to rip all of my CDs and
DVD collection isn't exactly insignificant at this point, either.  Sure it
could be recreated from scratch if push came to shove, but I'd rather not go
through that again.

<!-- more -->

The cheapest way to backup your data these days of course seems to be to just
buy more disks, but with that comes the problem of what to plug them into.
Enter the HP N36L MicroServer.  It might only have a relatively meagre 1.3GHz
Athlon II Neo CPU (dual core though!) but it'll support up to 8GB of RAM and
most importantly, it has 4 x SATA drive bays as well as an additional SATA
header on board.  A few clicks on eBuyer.com led to this little lot:

* 1 x HP N36L MicroServer;
* 2 x 4GB DIMMs;
* 4 x 2TB Samsung HDDs

All for the princely and scarcely-believable sum of £434, once you factor in
HP's £100 cashback offer.  With a nicely compatible set of hardware onboard the
opportunity to double this up as my new vSphere 'lab' was too much to resist so
I installed and booted ESXi off a USB stick plonked into the header on the
motherboard.  I had a spare 750GB drive knocking around which I stashed in the
5.25" bay at the top and connected it to the spare SATA header, but what about
all those other disks?  That's where I started to scratch my head.

The MicroServer doesn't do "proper" RAID, the BIOS essentially fronts a
software-based solution which isn't supported by VMware.  You could import and
provision them with ESXi for use by all the virtual machines, but eventually I
decided to go down the route of raw-device mapping the disks directly to one
VM, and that machine would run FreeBSD so that I could take advantage of ZFS.
 The other upside to this is that if I decide for some reason to take ESX out
of the equation, I can boot and run FreeBSD natively and just import my zpool.
 I don't have to worry about the messiness of VMFS or anything else like that.

With that decision made and ESXi installed, the first step was to RDM the locally-attached drives to my VM.  This isn't supported via the VMware client, it has to be done from the commandline as an option to 'vmkfstools'.  The procedure is <a title="SATA RDMs in ESXi" href="http://www.vm-help.com/esx40i/SATA_RDMs.php">well document elsewhere</a> so I won't bother with the details here, but once that was done FreeBSD saw all four disks on boot and I created a new RAIDZ array with those four drives.  So now onto some performance testing - just how much does my setup suffer when it's virtualised?  Here's a real-world example, the time taken to rsync (-av) a ~30GB directory over NFS:

* Native

``` 
sent 30211839333 bytes  received 19108 bytes  4125330.57 bytes/sec
total size is 30208073684  speedup is 1.00
real    122m3.212s
user    6m11.601s
sys     5m4.499s
```

* Virtualised

``` 
sent 30211839333 bytes  received 19108 bytes  3530453.81 bytes/sec
total size is 30208073684  speedup is 1.00
real    142m37.140s
user    6m33.874s
sys     5m6.392s
```

So an extra 20 minutes to do the full 30GB, which isn't insignificant but I can
live with that given its purpose.  The native install had the run of the whole
8GB of RAM, but the virtualised instance had only 2GB allocated - and everyone
knows that ZFS loves RAM.  Aside from enabling ZFS prefetch for the VM, there
hasn't been much in the way of performance tuning here either - Jumbo Frames
are disabled, NFS is out-of-the box on both client and server in terms of
configuration, and so on.  Best do something about that then...
