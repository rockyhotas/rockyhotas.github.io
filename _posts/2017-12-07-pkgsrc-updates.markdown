---
layout: post
title:  "pkgsrc and package upgrades"
date:   2017-12-07 12:48:32 +0100
categories: netbsd
published: false
---
The [NetBSD][netbsd-link] package build (and distribution) system is
[pkgsrc][pkgsrc-link].

The base OS already provides the single tools `pkg_add`, `pkg_admin`, `pkg_info` (and
others) used to handle the pkgsrc packages. Anyway, a full manager can
be used instead: it is [`pkgin`][pkgin-link].

{% highlight bash %}
$ which vi
/usr/bin/vi

$ ls -l `find /etc/alternatives/vi -printf %l`
-rwxr-xr-x 1 root root 2932160 nov 24  2016 /usr/bin/vim.gnome
{% endhighlight %}

[netbsd-link]: http://www.netbsd.org
[pkgsrc-link]: http://www.pkgsrc.org
[pkgin-link]: http://pkgin.net
