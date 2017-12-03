---
layout: post
title:  "Different OSs, different vis"
date:   2017-12-03 18:25:32 +0100
categories: jekyll update
---
Unix-like Operating Systems can have different implementations of the same commands, utilities, and/or programs. One of the most remarkable cases is `vi`.

The Linux version of `vi` is actually `vim`. This can be verified as follows. This example has been run on Ubuntu 16.04:

{% highlight bash %}
$ which vi
/usr/bin/vi

$ ls -l `which vi`
lrwxrwxrwx 1 root root 20 giu 27  2016 /usr/bin/vi -> /etc/alternatives/vi

$ ls -l `find /usr/bin/vi -printf %l`
lrwxrwxrwx 1 root root 18 giu 27  2016 /etc/alternatives/vi -> /usr/bin/vim.gnome

$ ls -l `find /etc/alternatives/vi -printf %l`
-rwxr-xr-x 1 root root 2932160 nov 24  2016 /usr/bin/vim.gnome
{% endhighlight %}

Even if `which vi` returns immediately what can seem to be an executable, there is a matrioska-like structure of symlinks before reaching the *actual* executable file. The syntax `find /symlink/path -printf %l` is used to get the symlink target, as `which <name of program>` is used to get the path of the executable file.

The executable which is used in Ubuntu (and many other Linux flavours) to represent `vi` is `vim.gnome`, an edition of `VIM - Vi IMproved`.

This is not the case for [NetBSD][netbsd-link]. This OS uses the `nex/nvi` executable, which is (according to `man vi`) “as bug-for-bug compatible replacements for the original Fourth Berkeley Software Distribution (4BSD) ex and vi programs”.

[netbsd-link]: http://www.netbsd.org
