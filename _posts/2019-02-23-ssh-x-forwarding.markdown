---
layout: post
title:  "ssh X forwarding"
date:   2019-02-23 16:25:32 +0100
categories: unix
---
An `ssh` connection with X forwarding enabled lets the server to run a GUI application locally, and to display its graphical interface remotely, on the *client*'s screen. For example:

    user1@clienthost:~$ ssh -X user2@serverhost
    
    No mail.
    Last login: Sat Feb 23 12:07:43 2019 from 192.168.1.4
    user2@serverhost:~$ gedit &
    [1] 18102
    user2@serverhost:~$

This way, `gedit` is opened and `user1` can use it *theoretically* as if he had launched it locally, inside `clienthost`. Actually instead, the experience with this `gedit` graphic interface may be very unpleasant, showing a slow responsiveness and continuous lags.

This happens, for example, when one between `clienthost` and `serverhost` uses a Wi-Fi connection at 54 Mbps, or even an Ethernet powerline adapter at about 60 or 70 Mbps. The lags start being mitigated and less annoying when the connection is instead over an Ethernet cable link at 100 Mbps.

Along with `-X`, it is therefore recommended to use another option when starting the `ssh` connection to server: `-C`. From the manpage `ssh(1)`:

    -C      Requests compression of all data (including stdin, stdout, stderr, and
            data for forwarded X11, TCP and UNIX-domain connections). The
            compression algorithm is the same used by gzip(1). Compression is
            desirable on modem lines and other slow connections.

Run `ssh -CX user2@serverhost` to use it. This compression significantly increases the communication between the two hosts: a decent usability of the graphic interface is then granted also in the above examples of unsatisfying connections.

<br>

Thanks to [leot] for this suggestion.


[leot]: http://www.netbsd.org/~leot/
