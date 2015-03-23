--- 
layout: post
title: "'Minimal' Solaris 10 and SunCluster 3.2 install in VMware Fusion"
date: 2010-3-20
comments: false
categories: General Geek Solaris VMware
link: false
---

Following on from [Dan's how-to](http://thebsdbox.co.uk/?p=160) regarding the
configuration of SunCluster 3.2 within VMware Fusion, here's my additional
notes on doing this from a minimal install (i.e core software group) so as to
keep the size of the VMs down.

First thing you'll want to do is install some packages to make your life a bit
less painful.  Connect the DVD device with the Solaris 10 ISO and mount it
within your VM:

```
# mount -F hsfs -o ro /dev/dsk/c0d0t0s0 /mnt/cdrom
```

Then we can install some packages from /mnt/cdrom/Solaris_10/Product.  Let's
start off with some basic utilities, bash, and online documentation (man
pages):

```
# yes | pkgadd -d . SUNWxcu4
# yes | pkgadd -d . SUNWadmfr SUNWadmfw
# yes | pkgadd -d . SUNWbash
# yes | pkgadd -d . SUNWdoc SUNWman
```

SSH is also quite useful:

```
# yes | pkgadd -d . SUNWsshr SUNWsshu SUNWsshdr SUNWsshdu SUNWsshcu
# /lib/svc/method/sshd -c
# svcadm enable ssh
```

The second command will create your RSA and DSA host keys, without this you
won't be able to connect.

Then there's a bunch of additional prerequisite packages you need to install
before SunCluster itself.  These are as follows:

```
SUNWtcatu
SUNWj3rt
SUNWj3dev
SUNWmfrun
SUNWxwrtl
SUNWxwice
SUNWxwplt
SUNWctpls
SUNWxwfnt
SUNWxwplr
```

You also need some Solaris Zones related gubbins otherwise you'll see various
errors pertaining to missing libraries such as libzonecfg.so.1:

```
SUNWzoneu
SUNWzoner
SUNWpool
SUNWluu
SUNWluzone
SUNWlur
SUNWpoolr
SUNWlucfg
```

Finally, the core / reduced networking group installation also disables some
RPC related services that are necessary for cluster node communication. You’ll
find that installation hangs, for example, when the primary node is waiting for
the secondary to reboot. To remedy this, do the following:

```
# svccfg -s rpc/bind setprop config/local_only = false
# svccfg -s rpc/bind setprop config/enable_tcpwrappers = false
# svccfg -s rpc/bind listprop
# svcadm refresh rpc/bind
```

One other oddity I noticed was that I was seeing a bunch of RPC timeouts when
looking at anything IPMP related.  This turned out to be some DNS resolution
going on as a result of the IP addresses assigned to the cluster interconnects
and my ISP returning bullshit records.  It's easily fixed by editing
/etc/nsswitch.conf and amending the ipnodes line as follows:

```
ipnodes: files
```

You should then be ready to kick off the install of SunCluster, and everything
else is exactly as per Dan's guide.
