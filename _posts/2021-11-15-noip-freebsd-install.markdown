---
layout: post
title:  "no-ip FreeBSD install"
date:   2021-11-15 22:39:10 +0100
categories: unix, rc.d, freebsd
---
If you [ISP][isp] does not provide a static IP address to your modem, a Dynamic DNS service may be very useful. [no-ip.com][no-ip] provides this service for free.

As any DDNS, it needs an executable installed in a local machine, which regularly checks the public IP address of the modem (almost all the modems now integrate this service in their firmware, but this would be less funny). [no-ip.com][no-ip] offers this executable for Linux systems. It can be easily adapted to the FreeBSD build environment and installed; as an alternative, the [FreeBSD `pkg` repository][pkg-repository] already includes a package with an up-to-date version of this program, which also provide an `rc.d` script to use it as a daemon. In fact, this is exactly what DDNS would need: an executable constantly running.

Install it with `pkg install noip`. The actual executable is `/usr/local/bin/noip2`: the first time you run it (`/usr/local/bin` should be in you `$PATH`, so it should be enough to simply run `noip2`), the configuration is created through some simple guided steps. One the the parameters is the time interval between each public IP check. The configuration is stored by default in `/usr/local/etc/no-ip2.conf`.

This executable can be run as a daemon simply appending the line `noip_enable="YES"' in `/etc/rc.conf`, according to the install message. However, the `rc.d` script (which runs `noip2` as a daemon) comes with a major flaw: it can not start.

    # service noip start
    Starting noip.
    Can't locate configuration file /usr/local/etc/no-ip2.conf. (Try -c). Ending!

    /usr/local/etc/rc.d/noip: WARNING: failed to start noip

This error message apparently makes no sense, because:

    # ls -l /usr/local/etc/no-ip2.conf
    -rw-------  1 root  wheel  140 Nov 13 00:02 /usr/local/etc/no-ip2.conf

and because the `rc.d` script uses it, as it can be deduced by its default values, which are not overridden by any setting in `/etc/rc.conf` (the only variable modified by `/etc/rc.conf` is `noip_enable`, which is set to `YES`). Reading the script, located in `/usr/local/etc/rc.d/noip`, these values are in fact:

    : ${noip_config="/usr/local/etc/no-ip2.conf"}
    : ${noip_flags="-c $noip_config"}

However, the script includes other non-negligible defaults:

    : ${noip_user="noip"}
    : ${noip_group="noip"}

The installation from `pkg` does not create this user, nor this group. The script runs as user `noip`, which possibly translates into `nobody` in the system, and it has no access to the configuration file: this finally explains the error message.

Several options are available:

- run the `rc.d` script as privileged user, `root`, set `noip_user` to `root` and `noip_group` to `wheel`; they are already the user and group owner of both the executable `/usr/local/bin/noip2` and the configuration file `/usr/local/etc/no-ip2.conf`;
- create the non privileged user `noip` and the group `noip`; change the user and group owners of `/usr/local/bin/noip2` and `/usr/local/etc/no-ip2.conf`, so that the above defaults are satisfied;
- change the values of `noip_user` to another existing username, as well as `noip_group` to an existing group; change the user and group owners of `/usr/local/bin/noip2` and `/usr/local/etc/no-ip2.conf` accordingly.

It is recommended to run `noip2` as a non privileged user, so the second and third solutions are preferable.


[no-ip]: https://www.noip.com/
[isp]: https://en.wikipedia.org/wiki/Internet_service_provider
[pkg-repository]: https://www.freshports.org/
