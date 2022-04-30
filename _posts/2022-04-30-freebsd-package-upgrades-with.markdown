---
layout: post
title:  "FreeBSD package upgrades with non compatible kernel"
date:   2022-04-30 15:24:00 +0200
categories: freebsd, kernel, packages, updates
---
An example of upgrade of packages in FreeBSD is:

```
# pkg update
Updating FreeBSD repository catalogue...
Fetching packagesite.pkg: 100%    6 MiB   3.4MB/s    00:02
Processing entries: 100%
FreeBSD repository update completed. 31257 packages processed.
All repositories are up to date.
You have new mail.
# pkg upgrade
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Checking for upgrades (7 candidates): 100%
Processing candidates (7 candidates): 100%
The following 7 package(s) will be affected (of 0 checked):

Installed packages to be UPGRADED:
        ffmpeg: 4.4.2,1 -> 4.4.2_1,1
        libavif: 0.9.3_3 -> 0.9.3_4
        mlt7: 7.4.0_2 -> 7.4.0_3
        mlt7-qt5: 7.4.0_2 -> 7.4.0_3
        p5-subversion: 1.14.1 -> 1.14.2
        subversion: 1.14.1 -> 1.14.2
        svt-av1: 0.9.1_1 -> 1.0.0

Number of packages to be upgraded: 7

16 MiB to be downloaded.

Proceed with this action? [y/N]: y
[1/7] Fetching svt-av1-1.0.0.pkg: 100%    2 MiB   2.0MB/s    00:01
[2/7] Fetching subversion-1.14.2.pkg: 100%    3 MiB   3.3MB/s    00:01
[3/7] Fetching p5-subversion-1.14.2.pkg: 100%    1 MiB   1.1MB/s    00:01
[4/7] Fetching mlt7-qt5-7.4.0_3.pkg: 100%   92 KiB  94.4kB/s    00:01
[5/7] Fetching mlt7-7.4.0_3.pkg: 100%  595 KiB 609.0kB/s    00:01
[6/7] Fetching libavif-0.9.3_4.pkg: 100%   99 KiB 101.4kB/s    00:01
[7/7] Fetching ffmpeg-4.4.2_1,1.pkg: 100%    9 MiB   2.5MB/s    00:04
Checking integrity... done (0 conflicting)
[1/7] Upgrading svt-av1 from 0.9.1_1 to 1.0.0...
[1/7] Extracting svt-av1-1.0.0: 100%
[2/7] Upgrading ffmpeg from 4.4.2,1 to 4.4.2_1,1...
[2/7] Extracting ffmpeg-4.4.2_1,1: 100%
[3/7] Upgrading subversion from 1.14.1 to 1.14.2...
[3/7] Extracting subversion-1.14.2: 100%
[4/7] Upgrading mlt7 from 7.4.0_2 to 7.4.0_3...
[4/7] Extracting mlt7-7.4.0_3: 100%
[5/7] Upgrading p5-subversion from 1.14.1 to 1.14.2...
[5/7] Extracting p5-subversion-1.14.2: 100%
[6/7] Upgrading mlt7-qt5 from 7.4.0_2 to 7.4.0_3...
[6/7] Extracting mlt7-qt5-7.4.0_3: 100%
[7/7] Upgrading libavif from 0.9.3_3 to 0.9.3_4...
[7/7] Extracting libavif-0.9.3_4: 100%
Generating gdk-pixbuf modules cache
```

This is a successful procedure. Sometimes instead an error message of this kind is printed:

```
$ uname -a
FreeBSD myhostname 12.2-RELEASE-p4 FreeBSD 12.2-RELEASE-p4 GENERIC  amd64

# pkg update
Updating FreeBSD repository catalogue...
Fetching packagesite.pkg: 100%    6 MiB   3.4MB/s    00:02
Processing entries:   0%
Newer FreeBSD version for package p5-Tree-Parser:
To ignore this error set IGNORE_OSVERSION=yes
- package: 1203000
- running kernel: 1202000
Ignore the mismatch and continue? [y/N]: N
pkg: repository FreeBSD contains packages for wrong OS version: FreeBSD:12:amd64
Processing entries: 100%
Unable to update repository FreeBSD
Error updating repositories!
```

This error can also arise when a new package is directly installed, without trying the `pkg update`, `pkg upgrade` sequence.

According to the error, the package(s) in the remote repository have been built for another version of the FreeBSD kernel, which is newer than the one in the system. In the above example, the repository was created for the kernel `12.3`, while the current system was `12.2`. When a `RELEASE` (like `12.2`) is ended, all the packages switch to the next version and are built for that one (like `12.3`). For such little differences, there *could* be no issues (however, beware that the OS keeps being updated and the difference with the local version of the kernel becomes wider with time). However, any problem is avoided if the system is aligned to the kernel version of the repository. 

To upgrade the whole OS to version `12.3` of kernel, the first operation is:

```
# freebsd-update -r 12.3 upgrade
src component not installed, skipped
Looking up update.FreeBSD.org mirrors... 2 mirrors found.
Fetching metadata signature for 12.2-RELEASE from update2.freebsd.org... done.
Fetching metadata index... done.
Fetching 2 metadata patches.. done.
Applying metadata patches... done.
Fetching 1 metadata files... done.
Inspecting system... done.

The following components of FreeBSD seem to be installed:
kernel/generic kernel/generic-dbg world/base world/doc world/lib32

The following components of FreeBSD do not seem to be installed:
world/base-dbg world/lib32-dbg

Does this look reasonable (y/n)? y

Fetching metadata signature for 12.3-RELEASE from update2.freebsd.org... done.
Fetching metadata index... done.
Fetching 1 metadata patches. done.
Applying metadata patches... done.
Fetching 1 metadata files... done.
Inspecting system... done.
Fetching files from 12.2-RELEASE for merging... done.
Preparing to download files... done.
Fetching 10328 patches.....10....20....30....40....50....60....70....80....90....100....110....120....130....140....150....160....170....180....190....200....210....220..
..230....240....250....260....270....280....290....300....310....320....330....340....350....360....370....380....390....400....410....420....430....440....450....460....
470....480....490....500....510....520....530....540....550....560....570....580....590....600....610....620....630....640....650....660....670....680....690....700....71
0....720....730....740....750....760....770....780....790....800....810....820....830....840....850....860....870....880....890....900....910....920....930....940....950.
...960....970....980....990....1000....1010....1020....1030....1040....1050....1060....1070....1080....1090....1100....1110....1120....1130....1140....1150....1160....117
0....1180....1190....1200....1210....1220....1230....1240....1250....1260....1270....1280....1290....1300....1310....1320....1330....1340....1350....1360....1370....1380.
...1390....1400....1410....1420....1430....1440....1450....1460....1470....1480....1490....1500....1510....1520....1530....1540....1550....1560....1570....1580....1590...
```

The fetch continues until all the `10328` packages of this example are obtained:

```
...10280....10290....10300....10310....10320.... done.

Applying patches... done.

Fetching 166 files... ....10....20....30....40....50....60....70....80....90....100....110....120....130....140....150....160... done.

Attempting to automatically merge changes in files... done.
```

If the system is particularly out-of-date, it may happen that this first step seems to end correctly, but a reboot is not requested and the next step (which will be `/usr/sbin/freebsd-update install`) quickly ends without actually installing anything. If this is the case, two update iterations must be performed. So, repeat the procedure running a second time `freebsd-update -r <release_number> upgrade`.

After the fetch, the differences between the new files (which are going to be installed) and the current ones are shown, if any:

```
The following changes, which occurred between FreeBSD 12.2-RELEASE and

FreeBSD 12.3-RELEASE have been merged into /etc/ntp.conf:

--- current version

+++ new version

@@ -1,7 +1,7 @@

 #

-# $FreeBSD: releng/12.2/usr.sbin/ntp/ntpd/ntp.conf 352865 2019-09-29 03:36:50Z cy $

+# $FreeBSD: releng/12.3/usr.sbin/ntp/ntpd/ntp.conf 365704 2020-09-14 01:20:57Z emaste $

 #

 # Default NTP servers for the FreeBSD operating system.

 #

 # Don't forget to enable ntpd in /etc/rc.conf with:

 # ntpd_enable="YES"

@@ -12,12 +12,12 @@



 #

 # Set the target and limit for adding servers configured via pool statements

 # or discovered dynamically via mechanisms such as broadcast and manycast.

 # Ntpd automatically adds maxclock-1 servers from configured pools, and may

-# add as many as maxclock*2 if necessary to ensure that at least minclock

-# servers are providing good consistant time.

+# add as many as maxclock*2 if necessary to ensure that at least minclock

+# servers are providing good consistent time.

 #

 tos minclock 3 maxclock 6



 #

 # The following pool statement will give you a random set of NTP servers

Does this look reasonable (y/n)?
```

Then, lists of files which will (respectively) be removed, added and updated are possibly shown (they may be very long: press `j` to scroll down the list and `q` to exit):

```
The following files will be removed as part of updating to
12.3-RELEASE-p5:
/boot/lua/logo-beastie.lua
...
```

```
The following files will be added as part of updating to
12.3-RELEASE-p5:
/boot/kernel/ibcore.ko
...
```

```
The following files will be updated as part of updating to
12.3-RELEASE-p5:
/.cshrc
/.profile
/COPYRIGHT
/bin/[
/bin/cat
...
```

Then, a final instruction message is given:

```
To install the downloaded upgrades, run "/usr/sbin/freebsd-update install".
```

Run it and a reboot may be required:

```
# /usr/sbin/freebsd-update install
src component not installed, skipped
Installing updates...
Kernel updates have been installed.  Please reboot and run
"/usr/sbin/freebsd-update install" again to finish installing updates.
```

After the reboot:

```
# /usr/sbin/freebsd-update install
src component not installed, skipped
Installing updates...Scanning //usr/share/certs/blacklisted for certificates...
Scanning //usr/share/certs/trusted for certificates...
Scanning //usr/local/share/certs for certificates...
 done.
```

Now, the system upgrade to `12.3` is completed. The packages can be upgraded as usual (with the sequence `pkg update`, `pkg upgrade`), with no more errors.

