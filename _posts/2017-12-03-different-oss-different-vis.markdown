---
layout: post
title:  "Different OSs, different vis"
date:   2017-12-03 18:25:32 +0100
categories: linux netbsd
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

Even if `which vi` returns immediately what seems to be an executable, there is a matrioska-like structure of symlinks before reaching the *actual* executable file. The syntax `find /symlink/path -printf %l` is used to get the symlink target, as `which <name of program>` is used to get the path of the executable file.

The executable which is used in Ubuntu (and many other Linux flavours) to represent `vi` is `vim.gnome`, an edition of `VIM - Vi IMproved`.

This is not the case for [NetBSD][netbsd-link]:

{% highlight bash %}
$ ls -l `which vi`
-r-xr-xr-x  3 root  wheel  465503 Mar 11  2017 /usr/bin/vi
{% endhighlight %}

First of all, there are no symlinks and `/usr/bin/vi` is immediately an executable. Anyway, even when the name `vi` itself is used, there is *no* unique version of this well-known program.

The `vi` executable used by NetBSD is also known as `nvi`: respectively, `nex` and `nvi` are “bug-for-bug compatible replacements for the original Fourth Berkeley Software Distribution (4BSD) ex and vi programs” (as specified in the `man vi` page).

Even if nowadays all the screens have huge resolutions, the old 80x25 character screen format is still relevant sometimes. This is the case of some text mailreaders, as well as some code writing guidelines. It can then be useful to set the text editor so that it automatically wraps the line while typing, when a limit column number is reached.

For example, in both the default `vim.gnome` and NetBSD `vi` it is possible to write the following on a brand new file:

    This is a line which is intended to be very long; it should be written with extreme freedom and no limitations upon its maximium length.

The text editor will not wrap and the line will be saved as it has been written.

Let's consider a limit length of 72 characters (in order to have some margin characters with respect to 80). Using `vim.gnome` and other `vim` versions, it is necessary to type

    :set textwidth=72

after having opened the file. Also the abbreviated form `tw` instead of `textwidth` is allowed. From now on, all the lines will automatically be wrapped when the 72nd column is reached. The above text would look like:

    This is a line which is intended to be very long; this line should be 
    written with extreme freedom and no limitations upon its maximim length.

When the *i* of *written* is typed, the whole word is moved to a new line: it is in fact the 73rd character. To keep this setting for all the next files, the line

    set textwidth=72

can be put in the `vim.gnome` configuration file (in the home directory), named `.vimrc`. The program is `VIM` and configuration files often end their names with `rc`, as if it was a convention: it is then easy to remember. Note that such a file can contain this line alone, if no other settings are needed. To verify that this option is active, type `:set textwidth` (without assigning a number) in `vim.gnome`, and its actual value will be shown in this form: `textwidth=72`.

It looks like this should work on NetBSD's `vi` too, but it will not. Typing `:set textwidth=72` will cause the following error:

    set: no textwidth option: 'set all' gives all option values.

The same character limit can still be applied, typing

    :set wraplen=72

or adding the line `set wraplen=72` in the configuration file (in the home directory), with the *not-so-trivial* name `.nexrc` (it refers to `nex`, mentioned before). The abbreviated form `wl` is equivalent to `wraplen`. This file can contain no more lines, if no other options are needed; the value can be verified by typing `:set wraplen` (again, without assigning a number). It will be displayed as: `wraplen=72`.

In both `vim.gnome` and `vi`, all the available options can be shown (as suggested above) with `:set all`.

[netbsd-link]: http://www.netbsd.org
