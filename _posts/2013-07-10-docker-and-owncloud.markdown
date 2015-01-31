---
layout: post
title: "Getting started with Docker - Deploying ownCloud"
date: 2013-07-10 21:53
comments: false
categories: General Geek
---

[Docker](http://docker.io) is another piece of virtualisation-related
technology that's very much 'in vogue' at the minute.  And for good reason -
most of which will be immediately apparent and familiar if you've ever worked
with FreeBSD's Jails or Solaris Zones.  It's billed as 'The Linux container
engine', with a primary goal of providing a framework for self-contained,
easily redistributable and deployable images - independantly of hardware,
language, operating system, and so on.  The upshot really is that it kind of
looks and works a lot like the aforementioned Jails or Zones, in that it's a
way of neatly isolating applications or processes within a self-contained
environment that doesn't require a full-blown hypervisor or the typical set of
dedicated virtual hardware in order to run.  It's built on top of a few key
Linux kernel features, and the Docker project provides the awesomesauce in
terms of tooling and framework to make managing and deploying this stuff in a
repeatable and easily transferrable manner a snap.

So what can I use it for, you ask?  Good question, and one which I hope to
explain with an example image that'll allow you to run your own container
hosting another popular bit of open-source software -
[Owncloud](http://owncloud.org).

## Getting Started

First things first and that's getting Docker installed.  I'll be using Ubuntu
12.04 in my example, so I suggest you follow the [existing (and excellent) documentation](http://www.docker.io/gettingstarted/) on getting Docker up and
running.  Once you've managed that we should be on the same page:

    # docker version
    Client version: 0.4.8
    Server version: 0.4.8
    Go version: go1.1

If you've followed the installation document correctly, you'll have a default
set of base images to work with as a starting point for a project such as this.
We can see what's available with the following:

    # docker images
    REPOSITORY          TAG                 ID                  CREATED             SIZE
    base                latest              b750fe79269d        3 months ago        24.65 kB (virtual 180.1 MB)
    base                ubuntu-12.10        b750fe79269d        3 months ago        24.65 kB (virtual 180.1 MB)
    base                ubuntu-quantal      b750fe79269d        3 months ago        24.65 kB (virtual 180.1 MB)
    ubuntu              12.10               b750fe79269d        3 months ago        24.65 kB (virtual 180.1 MB)
    ubuntu              latest              8dbd9e392a96        12 weeks ago        131.5 MB (virtual 131.5 MB)
    ubuntu              quantal             b750fe79269d        3 months ago        24.65 kB (virtual 180.1 MB)

As we'll be basing our Owncloud install around LTS, we need to pull in that
image specifically:

    # docker pull ubuntu:12.04

Once that command completes, `docker images` should include the following in its output:

    ubuntu               12.04               8dbd9e392a96        3 months ago        131.5 MB (virtual 131.5 MB)

Now we've got our starting point it's on to Owncloud itself.

## Owncloud

As you'd probably expect, the base image we've pulled in above is relatively
sparse and Owncloud has numerous dependancies including Apache httpd and PHP.
There's a few steps we need to run through before we're able to install the
Owncloud package, including grabbing the various dependancies and amending the
Apache configuration.  While it's possible to create a new container from the
base image and manually run the necessary commands, Docker has a concept known
as 'Dockerfiles' which allow you to essentially batch all these steps up and
have it run through them for you, delivering a built-to-spec image to use.  For
Owncloud, we'll need a Dockerfile with the following:

	FROM ubuntu:12.04
	MAINTAINER You "you@your.email"
	RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" >> /etc/apt/sources.list
	RUN apt-get -y update
	
	RUN dpkg-divert --local --rename --add /sbin/initctl
	RUN ln -s /bin/true /sbin/initctl
	
	RUN apt-get install -y apache2 php5 php5-gd php-xml-parser php5-intl php5-sqlite smbclient curl libcurl3 php5-curl bzip2 wget vim
	
	RUN wget -O - http://download.owncloud.org/community/owncloud-5.0.7.tar.bz2 | tar jx -C /var/www/
	RUN chown -R www-data:www-data /var/www/owncloud
	
	ADD ./001-owncloud.conf /etc/apache2/sites-available/
	RUN ln -s /etc/apache2/sites-available/001-owncloud.conf /etc/apache2/sites-enabled/
	RUN a2enmod rewrite
	
	EXPOSE :80
	
	CMD ["/usr/sbin/apache2ctl", "-D",  "FOREGROUND"]

Stick the above into a file called `Dockerfile`.  This should all look
relatively familiar but there's a few lines that warrant a bit more
explanation: 

* 6-7, which are needed because Upstart is currently broken in Docker.  This is a workaround so that apt's actions successfully complete;
* 18 tells Docker to NAT port 80 on our host to the container when it's run;
* 20 is the command that starts Apache's httpd in the foreground.  This is so that Docker can manage the process.

We also need to inject a simple configuration file for Apache into this image
so that it's included when httpd starts.  In the same folder that you've saved
this Dockerfile, create a new file called `001-owncloud.conf` with the
following basic directives:

    <Directory /var/www/owncloud>
    		Options Indexes FollowSymLinks MultiViews
    		AllowOverride All
    		Order allow,deny
    		allow from all
    </Directory>

## Creating an image

Now we're in a position to go ahead and create our Owncloud image.  Run the
following command then sit back and watch the magic:

    # docker build -t owncloud .

At this point Docker will run through the steps in your Dockerfile and show you
the status at each step:

    Uploading context 10240 bytes
    Step 1 : FROM ubuntu:12.04
     ---> 8dbd9e392a96
    Step 2 : MAINTAINER Nick Jones "nick@dischord.org"
     ---> Running in bd4ffa2bde51
     ---> 5be3042626e5
    Step 3 : RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" >> /etc/apt/sources.list
     ---> Running in 6342a8a5bf3a
     ---> 0a5f6332ccf1
    Step 4 : RUN apt-get -y update
     ---> Running in 5ba451710f4c
     ---> 25275d3b5080
    Step 5 : RUN dpkg-divert --local --rename --add /sbin/initctl
     ---> Running in 0a0473208260
     ---> 927abde6330d
    Step 6 : RUN ln -s /bin/true /sbin/initctl
     ---> Running in 92844ba5d028
     ---> 21680ab43143
    Step 7 : RUN apt-get install -y apache2 php5 php5-gd php-xml-parser php5-intl php5-sqlite smbclient curl libcurl3 php5-curl bzip2 wget vim
     ---> Running in 0380af0585d2
     ---> 2d2684e40291
    Step 8 : RUN wget -O - http://download.owncloud.org/community/owncloud-5.0.7.tar.bz2 | tar jx -C /var/www/
     ---> Running in f21f8a471b80
     ---> dc2c75b30292
    Step 9 : RUN chown -R www-data:www-data /var/www/owncloud
     ---> Running in b9b3e7220075
     ---> 74f5580c700b
    Step 10 : ADD ./001-owncloud.conf /etc/apache2/sites-available/
     ---> c0179ce255ef
    Step 11 : RUN ln -s /etc/apache2/sites-available/001-owncloud.conf /etc/apache2/sites-enabled/
     ---> Running in 6ba5513a8432
     ---> f96198e6b59d
    Step 12 : RUN a2enmod rewrite
     ---> Running in 161a90a9e940
     ---> 67cea873cdd7
    Step 13 : EXPOSE :80
     ---> Running in 9d4cfd17812a
     ---> 00bd10cd6f7d
    Step 14 : CMD ["/usr/sbin/apache2ctl", "-D",  "FOREGROUND"]
     ---> Running in 2ccef53a5cfe
     ---> f0d7dbd897c5
    Successfully built f0d7dbd897c5

The output of `docker images` should now include our freshly built Owncloud
image:

    # docker images | grep owncloud
    owncloud            latest              f0d7dbd897c5        About a minute ago   12.29 kB (virtual 779.4 MB)

## Running the container

With this built-to-spec image we're in a position to create a new container
which will run our Owncloud 'stack', isolated from the host OS and using only
the bare essentials:

    # docker run -d owncloud
    d7db4d7c6499

This creates and runs the new container; The `-d` switch tells Docker to kick
it off in the background.  If successful that command returns a UUID -
`d7db4d7c6499` in my case.  We can view running containers by doing the
following:

    # docker ps
    ID                  IMAGE               COMMAND                CREATED             STATUS              PORTS
    d7db4d7c6499        owncloud:latest     /usr/sbin/apache2ctl   2 minutes ago       Up 2 minutes        80->80  

Looks good so far.  If you now fire up your browser and point it at your Docker
host /owncloud/ you should be greeted by the initial Owncloud setup page asking
you to create an Admin user:

{% img center http://db.tt/vpKBHEmj %}

Go ahead and do that and you should now be logged into your new Owncloud
install, running atop Docker's various technologies.

## Managing containers

So we now have our Owncloud container and our base image, what's next?
Stopping the container is pretty straightforward:

    # docker stop d7db4d7c6499
    d7db4d7c6499
    # docker ps
    ID                  IMAGE               COMMAND             CREATED             STATUS              PORTS

The output of `docker ps` shows us that nothing is currently running.  To see all containers, stopped or otherwise, do `docker ps -a`:

    # docker ps -a
    ID                  IMAGE               COMMAND                CREATED             STATUS              PORTS
    d7db4d7c6499        owncloud:latest     /usr/sbin/apache2ctl   12 minutes ago      Exit 137                                
    2ccef53a5cfe        00bd10cd6f7d        /bin/sh -c #(nop) CM   15 minutes ago      Exit 0                                  
    9d4cfd17812a        67cea873cdd7        /bin/sh -c #(nop) EX   15 minutes ago      Exit 0                                  
    161a90a9e940        f96198e6b59d        /bin/sh -c a2enmod r   15 minutes ago      Exit 0                                  
    6ba5513a8432        c0179ce255ef        /bin/sh -c ln -s /et   15 minutes ago      Exit 0                                  
    [..]

I've snipped the output for brevity, but what's interesting here to note is
that Docker effectively created a new container from a new image for each step
in our Dockerfile before finally committing that image with the tag we
specified with the `-t` option.

To restart our container, run the following:

    # docker restart d7db4d7c6499
    d7db4d7c6499
    # docker ps
    ID                  IMAGE               COMMAND                CREATED             STATUS              PORTS
    d7db4d7c6499        owncloud:latest     /usr/sbin/apache2ctl   15 minutes ago      Up 18 seconds       80->80             

## Next Steps

That's hopefully given you a feel for Docker and the basics of how to get
something up and running.  At this point we have a very basic installation of
Owncloud, there's potentially a lot more to do before you'd consider making
full-time use of it, such as updating the Apache httpd configuration to enable
TLS and adding MySQL into the mix instead of SQLite as the database backend.

## tl;dr - Feeling lazy?

What if you don't care about creating your own image from scratch, you just
want to be able to run an Owncloud instance with a couple of commands?  From
what I've described so far, this technology should be all about being able to
do just that, right?  Yup, of course - just do the following:

    # docker pull yankcrime/owncloud

And subsequently spin up a new container from that image:

    # docker run -d owncloud

Easy.  Starting to understand the potential? (-:

Part 2 - adding SSL and MySQL into the mix - is [online here](http://dischord.org/blog/2013/08/13/docker-and-owncloud-part-2/).
