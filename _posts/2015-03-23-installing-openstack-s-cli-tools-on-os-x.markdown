---
layout: post
title: Installing OpenStack's CLI tools on OS X
date: 2015-03-23 12:01
published: true
categories:
---

Here's a quick guide for installing OpenStack's CLI tools on OS X in a way that doesn't make too much of a mess.

First up we need to get a couple of additional tools installed that will 'pollute' your base install, but once we're there we can easily create 'virtual' Python development environments and install isolated sets of packages on a per-project basis.  This includes creating an 'openstack' project in which we'll install keystone, glance, and so on.

	$ sudo easy_install pip
	$ sudo pip install virtualenv
	$ sudo pip install virtualenvwrapper

With those three things in place, we can then set about configuring a home directory for our virtual Python environments:

	$ mkdir .virtualenv
	$ source /usr/local/bin/virtualenvwrapper.sh

And now let's create an OpenStack Python development environment:

	$ mkvirtualenv openstack

To work on this environment at any time, do:

	$ workon openstack

At which point we can go ahead and install the various command-line tools and their dependances all within this virtual 'openstack' environment:

	$ (openstack)nick@deadline:~> pip install python-{keystone,glance,nova,neutron,cinder}client
	Downloading/unpacking python-keystoneclient
		Downloading python_keystoneclient-1.2.0-py2.py3-none-any.whl (392kB): 392kB downloaded
	Downloading/unpacking python-glanceclient
		Downloading python_glanceclient-0.17.0-py2.py3-none-any.whl (84kB): 84kB downloaded

	[..]

	Successfully installed python-keystoneclient python-glanceclient python-novaclient python-neutronclient oslo.serialization six oslo.utils oslo.config oslo.i18n msgpack-python netifaces

Done - all you have to remember is that each time you want to use these CLI tools you'll need to first do `workon openstack`.
