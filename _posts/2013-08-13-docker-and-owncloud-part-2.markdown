---
layout: post
title: "Deploying ownCloud with Docker, Part 2"
date: 2013-08-13 10:03
comments: true
categories: General Geek
---

If you followed my [previous guide](http://dischord.org/blog/2013/07/10/docker-and-owncloud/) (and haven't
done much with it since...), then you'll have a basic install of ownCloud
sitting in a Docker container as well as a corresponding base image.  Although
this is a good starting point, it's far from ideal for a number of reasons - we
haven't configured SSL nor have we configured a slightly more efficient
database backend (MySQL), for example. To get this to a point where we could
happily run it on a daily basis and trust our data with it, we need to do a
some more work.

<!-- more -->

Attempting to stay vaguely compliant with the Docker philosphy of being
host-agnostic and easily transportable means that we also need to get a little
bit clever about a few things. With this type of setup there are a few
environment-specific aspects to the configuration we need to be aware of, such
as what to use for the SSL certificate generation (notably the Common Name), as
well as passwords for things like MySQL's root account and the ownCloud
database that we'll create.

So with that in mind, let's start off by updating the Dockerfile - our starting
point for creating a Docker image:

```
FROM ubuntu:12.04
MAINTAINER Nick Jones "nick@dischord.org"
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" >> /etc/apt/sources.list
RUN apt-get -y update
 
RUN dpkg-divert --local --rename --add /sbin/initctl
RUN ln -s /bin/true /sbin/initctl
 
RUN locale-gen en_US en_US.UTF-8
RUN dpkg-reconfigure locales
 
RUN echo "mysql-server-5.5 mysql-server/root_password password root123" | debconf-set-selections
RUN echo "mysql-server-5.5 mysql-server/root_password_again password root123" | debconf-set-selections
RUN echo "mysql-server-5.5 mysql-server/root_password seen true" | debconf-set-selections
RUN echo "mysql-server-5.5 mysql-server/root_password_again seen true" | debconf-set-selections
 
RUN apt-get install -y apache2 php5 php5-gd php-xml-parser php5-intl php5-sqlite mysql-server-5.5 smbclient curl libcurl3 php5-mysql php5-curl bzip2 wget vim openssl ssl-cert
 
RUN wget -q -O - http://download.owncloud.org/community/owncloud-5.0.10.tar.bz2 | tar jx -C /var/www/
 
RUN mkdir /etc/apache2/ssl
 
ADD resources/cfgmysql.sh /tmp/
RUN chmod +x /tmp/cfgmysql.sh
RUN /tmp/cfgmysql.sh
RUN rm /tmp/cfgmysql.sh
 
ADD resources/001-owncloud.conf /etc/apache2/sites-available/
ADD resources/start.sh /start.sh
 
RUN ln -s /etc/apache2/sites-available/001-owncloud.conf /etc/apache2/sites-enabled/
RUN a2enmod rewrite ssl
 
EXPOSE :443
 
RUN chown -R www-data:www-data /var/www/owncloud
```

There's a few key changes here that I should explain before we go much further:

* Lines 12-15 set some debconf preferences for our MySQL installation ahead of
  time.  You'll want to change the password here to something more appropriate, ensuring that MySQL is 
  secured as the packages are installed;
* Line 17 now includes the various MySQL-related server packages;
* Line 19 is grabbing the latest version of the ownCloud tarball from the site;
* Resources being added from a 'resources' folder.

So what do we do about our MySQL configuration and customisation for ownCloud?
Well, line 20's resources/cfgmysql.sh script that we'll insert contains the
following:

```
#!/bin/bash
/usr/bin/mysqld_safe &
sleep 5
/usr/bin/mysql -u root -proot123 -e "CREATE DATABASE owncloud; GRANT ALL ON owncloud.* TO 'owncloud'@'localhost' IDENTIFIED BY 'owncloudsql';"
```

Amend the -p option to use the MySQL root password specified above and you
should also edit the 'owncloudsql' password to be something a little more
secure.

The 'sleep 5' is necessary in order to give MySQL enough time to start before
creating the database and assigning permissions.  Note that we can't have this
as seperate 'RUN' directives in our Dockerfile as, if you'll recall, Docker
starts a new container per command and then commits the change if the command
is successful.  This way we know that 'mysqld' is running before attempting to
run in our bit of SQL.

Next up are the SSL aspects of our configuration.  Line 22 of this Dockerfile
takes care of including a second script into our install - start.sh - and it's
in here that we'll do the necessary post-container-creation customisation to
get things like our SSL certificates for Apache correctly generated.  Let's
take a closer look at this:

```
#!/bin/bash
if [ ! -f /etc/apache2/ssl/server.key ]; then
        mkdir -p /etc/apache2/ssl
        KEY=/etc/apache2/ssl/server.key
        DOMAIN=$(hostname)
        export PASSPHRASE=$(head -c 128 /dev/urandom  | uuencode - | grep -v "^end" | tr "\n" "d")
        SUBJ="
C=UK
ST=England
O=Dischord
localityName=Manchester
commonName=$DOMAIN
organizationalUnitName=
emailAddress=nick@dischord.org
"
        openssl genrsa -des3 -out /etc/apache2/ssl/server.key -passout env:PASSPHRASE 2048
        openssl req -new -batch -subj "$(echo -n "$SUBJ" | tr "\n" "/")" -key $KEY -out /tmp/$DOMAIN.csr -passin env:PASSPHRASE
        cp $KEY $KEY.orig
        openssl rsa -in $KEY.orig -out $KEY -passin env:PASSPHRASE
        openssl x509 -req -days 365 -in /tmp/$DOMAIN.csr -signkey $KEY -out /etc/apache2/ssl/server.crt
fi

HOSTLINE=$(echo $(ip -f inet addr show eth0 | grep 'inet' | awk '{ print $2 }' | cut -d/ -f1) $(hostname) $(hostname -s))
echo $HOSTLINE >> /etc/hosts

/usr/bin/mysqld_safe &
/usr/sbin/apache2ctl -D FOREGROUND
```

First off we do a quick test to see if this is being run for the first time in
a new container from an image, and if so generates our SSL keys accordingly.
The section to be customized here is what's in the SUBJ variable (but mind the
formatting!).  This allows for a certain degree of repeatability, i.e from the
one Docker image you could spin up multiple ownCloud containers and know that
they'll receive unique certificates.  

Note that here we're assuming the CN for the SSL certificate is the container's
hostname - more on that in a minute as we come to start it up.  The passphrase
is randomly generated but then stripped so that Apache can start without asking
for it.

Next, lines 23 and 24 of this start.sh script take care of populating
/etc/hosts with the hostname and IP address of the container.  This fixes a
problem with ownCloud to do with how it performs some checks against itself -
you might have [noticed an error](https://gist.github.com/mattwilliamson/6188354) with the version configured using my previous
guide.

Finally, the other file we will include from our resources/ folder is the
Apache configuration containing the necessary directives to enable SSL:

```
<Directory /var/www/owncloud>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
</Directory>

<VirtualHost *:443>
        DocumentRoot /var/www/owncloud
        <Directory /var/www/owncloud>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny
                allow from all
        </Directory>
        SSLEngine on
        SSLCertificateFile /etc/apache2/ssl/apache.crt
        SSLCertificateKeyFile /etc/apache2/ssl/apache.key
</VirtualHost>
```

And that should be all we need in terms of our Dockerfile build framework to
get our image created:

	# ls -R
	.:
	Dockerfile  resources
	./resources:
	001-owncloud.conf  cfgmysql.sh  start.sh

So let's kick off the build from this set of files and tag it as 'owncloud':

    # docker build -t owncloud .
	Uploading context 10240 bytes
	Step 1 : FROM ubuntu:12.04
	[..]
    Successfully built be56671aabdf

Now for the moment of truth - let's run up a container from our new image.
This is where we need to pass the hostname that the container will use with the
'-h' option:

    # OWNCLOUD=$(docker run -d -h "owncloud.int.dischord.org" owncloud)
	# docker logs $OWNCLOUD
	Generating RSA private key, 2048 bit long modulus
	.+++
	...................+++
	e is 65537 (0x10001)
	No value provided for Subject Attribute organizationalUnitName, skipped
	writing RSA key
	Signature ok
	subject=/C=UK/ST=England/O=Dischord/L=Manchester/CN=owncloud.dischord.org/emailAddress=nick@dischord.org
	Getting Private key
	[..]

The remainder of the output should be MySQL starting up, and finally Apache
which stays in the foreground for Docker to monitor its status.  Now if we
point our browser to our host we should see the obligatory self-signed SSL
certificate warning but then:

{% img center https://dl.dropboxusercontent.com/u/174303/owncloudssl1.png %}

Huzzah!  Click the 'Advanced' dropdown and populate the MySQL options with the details that we used in our cfgmysql.sh script earlier:

{% img center https://dl.dropboxusercontent.com/u/174303/owncloudssl2.png %}

Sorted!  

Finally I should say that all the files used in this example are available on [Github](https://github.com/yankcrime/dockerfiles/tree/master/owncloud), and feel
free to hit me up on [Twitter](http://twitter.com/yankcrime) or [App.net](http://app.net/yankcrime) if you need any help or have any suggestions / corrections.
