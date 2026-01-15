---
layout: post
title:  "Introduction to file flags in NetBSD and FreeBSD"
date:   2026-01-15 22:59:00 +0200
categories: freebsd
---
**Disclaimer**: this is not an authoritative guide; it is just my understanding of this topic.

Besides Unix® file permissions, NetBSD and FreeBSD support the use of _flags_. According to the [FreeBSD Handbook][handbook], they provide an additional layer of security and control. Flags can be now applied to both files and directories (the Handbook probably still refers to an older version of this functionality), both in NetBSD and FreeBSD. If flags are used, some actions on files may be forbidden even for the user `root`.

Flags are documented for NetBSD and FreeBSD respectively in the manpages [`chflags(1)`][chflags-alt] and [`chflags(1)`][chflags]. They introduce the available flags as a list of keywords, also specifying which user can add them. The available flags in FreeBSD are more than NetBSD, with more structured facilities. The base system has several files with flags by default, unlike NetBSD.

As shown in the flags list, not any user can add or remove a flag: sometimes, only the super-user can do so, sometimes also the user owner of the file. Also note that (from FreeBSD [`chflags(1)`][chflags]):

```
The ability to change certain flags is dependent on the current kernel securelevel setting.
```

Directories are not directly mentioned neither in FreeBSD [`chflags(1)`][chflags], nor in NetBSD [`chflags(1)`][chflags-alt], despite flags can also be applied to them. It is probably assumed that directories are special kinds of files.

To add the flag `uappnd` to `examplefile`,

```
$ chflags uappnd examplefile
```

In both NetBSD and FreeBSD, if the above command is run on a file which already has some other flags set, `uappnd` is added to them: they are not replaced by `uappnd` only, but **this instead happens when the octal notation is used**, in both systems.

```
$ chflags 10 examplefile
```

will replace the flags of `examplefile`, whatever they are, with `opaque` only.

If the flag `schg` is added to a _directory_, no one (not even `root`, as anticipated) can modify it by creating new files or directories inside, or modifying the existing ones, regardless of the permissions.

To remove a flag, the same syntax is used, but with a `no` before the flag name:

```
$ chflags nouappnd examplefile
```

There is one exception (in both the OSs, even if in NetBSD this is undocumented): the flag `nodump` is removed by _removing_ (instead of adding) `no` from the flag name.

```
$ chflags dump examplefile
```

To remove all the flags of a file at once, it is more convenient to use the octal value:

```
# chflags 0 examplefile
```

This is valid both in NetBSD (where it is undocumented) and in FreeBSD (see [`chflags(1)`][chflags] for more information).

In both the OSs, to show the possible flags of a file, use `ls` with option `-o`. For example:

```
# ls -lo
total 8
-rw-r--r--  1 root wheel -                  1213 Nov 28 04:43 examplefile1
-rw-r--r--  1 root wheel sappnd,schg,nodump  806 Jan 11 23:42 examplefile2
```

Files with no flags will have a dash `-` in the fifth field, where otherwise flags are shown in a comma-separated list.

In addition to [`chflags(1)`][chflags], flags are also documented in [`find(1)`][find]. In FreeBSD,

```
       -flags [-|+]flags,notflags
	       The  flags are specified	using symbolic names (see chflags(1)).
	       Those with the "no" prefix (except "nodump")  are  said	to  be
	       notflags.   Flags  in flags are checked to be set, and flags in
	       notflags	are checked to be not set.  Note that this is  differ-
	       ent from	-perm, which only allows the user to specify mode bits
	       that are	set.

	       If  flags  are preceded by a dash ("-"),	this primary evaluates
	       to true if at least all of the bits in flags and	 none  of  the
	       bits  in	 notflags  are set in the file's flags bits.  If flags
	       are preceded by a plus ("+"), this primary evaluates to true if
	       any of the bits in flags	is set in the file's  flags  bits,  or
	       any  of	the  bits  in  notflags	is not set in the file's flags
	       bits.  Otherwise, this primary evaluates	to true	if the bits in
	       flags exactly match the file's flags  bits,  and	 none  of  the
	       flags bits match	those of notflags.
```

This may also be used to better understand the NetBSD [`find(1)`][find-alt], which is less clear as regards the common features. It can also be deduced that flags are stored in the filesystem as bit values in some defined positions, just as permissions or `setuid`, `setgid` and the sticky bit. This is the reason why a single flag can also be (in FreeBSD and also in NetBSD, undocumented) provided as an octal value.

Some examples and clarifications may be useful, or necessary. To find files with the flag `schg`:

```
$ find . -type f -flags schg
```

The manpage deals with a plural `flags`: if more than one, they must be provided in a comma-separated list. This is stated in [`chflags(1)`][chflags] for `chflags` itself (and then it is probably implicitly assumed for any other context where one or multiple flags can be specified, as in [`find(1)`][find]):

```
The flags are specified as an octal number or a comma separated list of keywords.
```

Consider the directory of the previous example:

```
# ls -lo
total 8
-rw-r--r--  1 root wheel -                  1213 Nov 28 04:43 examplefile1
-rw-r--r--  1 root wheel sappnd,schg,nodump  806 Jan 11 23:42 examplefile2
```

[`find(1)`][find-alt] states that, if the flag list is **not** preceded by a dash `-`, the `find` command evaluates to true only if the file has exactly the specified flags (not just _some_ of them and _not more_ than them), regardless of the order they are specified in `find` by the user.

```
# find . -type f -flags schg
(no results)
# find . -type f -flags schg,sappnd
(no results)
# find . -type f -flags schg,sappnd,nodump
./examplefile2
```

Otherwise, if the flag list is preceded by a dash `-`, the `find` command evaluates to true if the flags specified in `find` by the user are _between_ the flags of the file (and the file _may_ have other flags). However, note that this **doesn't** evaluate to true if the list includes the flags of the file _and_ other ones. In other words: it is true when the flags of the list are a subset of the flags of the file (and when they are the same, they are a subset which coincides with the original set), but it is not true when the flags of the file are a subset of the flags of the list.

```
# find . -type f -flags -schg
./examplefile2
# find . -type f -flags -schg,sappnd
./examplefile2
# find . -type f -flags -schg,sappnd,nodump
./examplefile2
# find / -type f -flags -arch,opaque,nodump,sappnd,schg,uappnd,uchg
(no results)
```

The last attempt gave no results, not even `./examplefile2`, because its flags are a subset of this list (the list contains more flags than the ones of the file).

Instead, if the flag list is preceded by a plus `+`, the last attempt would give `./examplefile2` as a result, because in this case (with respect to the previous one) `find` evaluates to true _even_ when the flags of the file are a subset of the flags of the list.

```
# find . -type f -flags +schg
./examplefile2
# find . -type f -flags +schg,sappnd
./examplefile2
# find . -type f -flags +schg,sappnd,nodump
./examplefile2
# find / -type f -flags +arch,opaque,nodump,sappnd,schg,uappnd,uchg
./examplefile2
```

In NetBSD, [`find(1)`][find-alt] more laconically states:

```
     -flags [-]flags
             If flags are preceded by a dash (``-''), this primary evaluates
             to true if at least all of the bits in flags are set in the
             file's flags bits.  If flags are not preceded by a dash, this
             primary evaluates to true if the bits in flags exactly match the
             file's flags bits.  If flags is ``none'', files with no flags
             bits set are matched.  (See chflags(1) for more information about
             file flags.)
```

The manpage [`find(1)`][find-alt] doesn't consider the case of multiple flags, but they can be specified in `find`, and in this case they must be in a comma-separated list. This is stated in [`chflags(1)`][chflags-alt]:

```
Flags are a comma separated list of keywords.
```

Here, `none` can be specified as a flag in [`find(1)`][find-alt] (but not in [`chflags(1)`][chflags-alt]), so

```
# find / -type f -not -flags none
```

will find all the files having at least one active flags. On the other hand, `find / -type f -flags none` will find all the files without flags. In a NetBSD 10.1 fresh installations, these are all the files in the system, because by default there are no files with flags set. Note that in NetBSD `none` can not be used by `chflag`. In other words, `chflags none somefile` is not accepted and therefore it can not be used to clear all the flags of a file.

Apparently, in FreeBSD there is instead no direct way to use `find(1)` and detect files having _any_ flag, that is: print all the files having _at least_ one flag. A tricky and lengthy syntax seems the only way to achieve this, using all the available flags one by one, preceded by dashes. As regards the flags with two or more equivalent keywords, it is enough to specify just one of them. For example, `-flags uappnd` will find all the files with the flag `uappnd`, even if it has been added as `uappend`, and vice-versa. Run this as `root`, to be sure to consider all the files and all the flags:

```
# find / -type f -flags -arch -or -flags -nodump -or -flags -opaque -or -flags -sappnd -or -flags -schg -or -flags -snapshot -or -flags -sunlnk -or -flags -uappnd -or -flags -uarch -or -flags -uchg -or -flags -uhidden -or -flags -uoffline -or -flags -urdonly -or -flags -usparse -or -flags -usystem -or -flags -ureparse -or -flags -uunlnk
```

The results in a FreeBSD 15 fresh installation are:

```
/usr/bin/chpass
/usr/bin/crontab
/usr/bin/login
/usr/bin/passwd
/usr/bin/su
/usr/lib32/libc.so.7
/usr/lib32/libcrypt.so.5
/usr/lib32/librt.so.1
/usr/lib32/libsys.so.7
/usr/lib32/libthr.so.3
/boot/entropy
/lib/libc.so.7
/lib/libcrypt.so.5
/lib/librt.so.1
/lib/libsys.so.7
/lib/libthr.so.3
/libexec/ld-elf.so.1
/libexec/ld-elf32.so.1
/sbin/init
/var/db/entropy/saved-entropy.1
/var/db/entropy/saved-entropy.2
/var/db/entropy/saved-entropy.3
/var/db/entropy/saved-entropy.4
/var/db/entropy/saved-entropy.5
/var/db/entropy/saved-entropy.6
/var/db/entropy/saved-entropy.7
/var/empty
/entropy
/.sujournal
```

[handbook]: https://docs.freebsd.org/en/books/handbook/basics/
[chflags]: https://man.freebsd.org/cgi/man.cgi?query=chflags&apropos=0&sektion=0&manpath=FreeBSD+15.0-RELEASE+and+Ports&arch=default&format=html
[chflags-alt]: https://man.netbsd.org/chflags.1
[find]: https://man.freebsd.org/cgi/man.cgi?query=find&apropos=0&sektion=0&manpath=FreeBSD+15.0-RELEASE+and+Ports&arch=default&format=html
[find-alt]: https://man.netbsd.org/find.1
