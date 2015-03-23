--- 
layout: post
title: Multiplexing SSH and SSL
date: 2010-8-25
comments: false
categories: General
link: false
---

It's fairly common to want to run SSH on port 443 these days, especially if you
need to connect to your server from behind a restrictive company firewall that
only allows egress connections to port 80 and 443.  This is all well and good,
but what happens when you want to use SSL/TLS on your webserver as well?

This is where [sslh](http://www.rutschle.net/tech/sslh.shtml) comes in.  It's a
really simple tool that wraps incoming connections to these ports and then
depending on protocol redirects it onto sshd back on port 22, or to your httpd
on localhost:443.  With me so far?  Good - it's easier to configure than it is
to explain ;)

<!--more-->

In this example, I compiled sslh from source and configured it manually in
conjunction with [Lighttpd](http://lighttpd.org)  First up, grab the latest
version from sslh's homepage:

```
$ wget http://www.rutschle.net/tech/sslh-1.7a.tar.gz
$ tar zxvf sslh-1.7a.tar.gz
$ cd sslh-1.7a
```

Take a quick look at the Makefile and amend the option for libwrap if you don't
want to use it.  Then it's simple a case of running:

```
$ make && make install
```

This should drop the binary in /usr/local/sbin (unless you changed this in the
Makefile!), the default configuration in /etc/default/sslh, and an appropriate
init script in /etc/init.d/sslh.  Now all we have to do is amend the
configuration and update it to reflect our host's settings.  In my case, I
replaced the 'ifname' with dischord.org's IP address:

```
LISTEN=212.13.197.13:443
SSH=localhost:22
SSL=localhost:443
```

Now we have to amend our webserver's configuration.  In my example, I'm using
Lightttpd but it's just as straightforward for Apache's httpd.  Firstly, we'll
want to quickly create a self-signed certificate:

```
$ sudo mkdir /etc/lighttpd/certs ; cd /etc/lighttpd/certs
$ sudo openssl req -new -x509 -keyout lighttpd.pem -out lighttpd.pem -days 3650 -nodes
$ sudo chmod 400 /etc/lighttpd/certs/lighttpd.pem
```

In Debian's case, they split out Lighttpd's configuration into a
/etc/lighttpd/conf-available directory.  You then just symlink in the relevant
bits that you need into /etc/lighttpd/conf-enabled.  So here we need to edit
the 10-ssl.conf and then enable it:

```
$ sudo vi /etc/lighttpd/conf-available/10-ssl.conf
$SERVER["socket"] == "127.0.0.1:443" {
                  ssl.engine                  = "enable"
                  ssl.pemfile                 = "/etc/lighttpd/certs/lighttpd.pem"
}
$ sudo ln -s /etc/lighttpd/conf-available/10-ssl.conf \
/etc/lighttpd/conf-enabled/10-ssl.conf
```

Here we've specified our newly-created and self-signed certificate, and told
Lighttpd's SSL engine that it should only bind to the localhost loopback
address on port 443.  Now we can start sslh and restart Lighttpd:

```
$ sudo /etc/init.d/sslh start
$ sudo /etc/init.d/lighttpd restart
```

If all's well, you should now be able to connect via SSH to port 443 on your
server, but also still use HTTPS in your web-browser when hitting your site.
