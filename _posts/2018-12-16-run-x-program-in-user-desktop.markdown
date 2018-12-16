---
layout: post
title:  "Run X program in another user's GUI"
date:   2018-12-16 14:59:32 +0100
categories: linux
---
System administrators (sysadmins, from now on) may occasionally need to display messages to the user who is currently running a [GUI][gui-wiki]: let his username be `targetuser`; let sysadm's username be `admguy`; let local system's hostname be `guihost`.

In [Linux][linux-wiki] and other *nix OSs, graphical interfaces are usually handled by [X Window System][x-window-system-wiki]. In this example, sysadm needs:

- to belong to the local system group `sudo`;
- to be connected to the local system through a text interface;
- to properly set up some environment variables.

Sysadmin does not need to know (he is also *not allowed* to know) `targetuser`'s password.

    admguy@guihost:~$ who
    targetuser	tty7	2018-12-16 12:02 (:0)
    admguy		pts/8	2018-12-16 14:53 (192.168.1.2)

This is an example output of `who` in the local system. Sysadm `admguy` is logged in through a remote [SSH][ssh-wiki] connection; `targetuser` has instead performed a graphic login and his GUI runs on local display `:0`.

Sysadmin may choose to act:

- as himself;
- as `targetuser`.

If acting by himself, sysadm must first set up his environment variable `DISPLAY` to match `targetuser`'s one: `:0`.

    admguy@guihost:~$ export DISPLAY=:0

Note that this variable was originally not defined in sysadm's [`env`][env-wiki], because he is using a text-only connection. [X Window System][x-window-system-wiki] grants access to a display only if some credentials are provided. When `targetuser` loaded GUI for the first time, a hidden file `.Xauthority` in his home directory `/home/targetuser` or equivalently `~targetuser` was created. Its contents can be shown as:

    targetuser@guihost:~$ xauth list
    guihost/unix:0  MIT-MAGIC-COOKIE-1  bd2b204d7d9b2328e4bfc3b2b7a237f0

X applications run by `targetuser` can connect to display `:0` through a [Unix socket][unix-socket-wiki] on host `guihost` only if they provide the correct `MIT-MAGIC-COOKIE-1` value, which is used for authentication. File `.Xauthority` for `targetuser` lists: hostname, socket type, display, cookie type and cookie value. GUI applications run by `targetuser` must read information on this file to be allowed to run on display `:0`, with `targetuser` as owner. Note that access to `.Xauthority` is very restrictive:

    targetuser@guihost:~$ ls -la | grep Xauthority
    -rw-------   1 targetuser targetuser        277 dic 16 12:02 .Xauthority

If sysadm wants to run a GUI application on `targetuser`'s display, this application must follow the same procedure, but -- being launched by sysadm -- it is only able to read sysadm's `.Xauthority` file, **not** `targetuser`'s one. However, sysadm can acquire the `~targetuser/.Xauthority` information into *his own* `.Xauthority` file: the canonical way is through X command `xauth`.

Information can be extracted from a specific `.Xauthority` file from its owner (which, in this case, is `targetuser`) as:

    targetuser@guihost:~$ xauth extract - $DISPLAY

See [`man xauth`][xauth-man] for more information (this is referred to the current most recent X11R7 release; see also the manpage on your Operating System). This will print some illegible characters on screen (because `stdout` has been chosen with `-` for output of `xauth extract`): they are suitable, though, to be read with `xauth merge -`. Note that only `targetuser` can perform `xauth extract`, because he is the only one who is able to read `~targetuser/.Xauthority` file; on the other hand, if sysadm must acquire this information, `xauth merge -` must be run by `admguy`. 

Sysadm must then act as `targetuser` when performing `xauth extract - $DISPLAY` (and `sudo` lets him to do this); then, he must act as **himself** to acquire that `~targetuser/.Xauthority` output information and merge it into `~admguy/.Xauthority`:

    admguy@guihost:~$ sudo -u targetuser -i xauth extract - $DISPLAY | xauth merge -

Acting as `targetuser` through `sudo`, information on `~targetuser/.Xauthority` is extracted to `admguy`'s `stdout`; then, with `xauth merge -` (which reads `stdout`) normally run as `admguy`, it is gathered into sysadms's `.Xauthority` file.

Sysadm `admguy` **is now able to act** on `targetuser`'s display as well as `targetuser`, because the environment variable `DISPLAY` is already set to `:0` and he owns the required authentication credentials. Then, for example,

    admguy@guihost:~$ xclock

displays a clock in `targetuser`'s GUI. Note that the process is owned by `admguy`, the sysadm user:

    admguy@guihost:~$ ps -aux | grep xclock
    admguy 23102  0.0  0.0  71048  6580 pts/8    S+   17:03   0:00 xclock

<br>

A non-canonical, alternative way is to copy `~targetuser/.Xauthority` through `sudo` to an appropriate location, change its owner from the default `root` (which is the owner after the `sudo` copy operation) to `admguy`, and set the value of the environmental variable `XAUTHORITY` to the **full path** of the copied `.Xautority` file:

    admguy@guihost:~$ export DISPLAY=:0
    admguy@guihost:~$ sudo cp ~targetuser/.Xauthority /custom_directory/custom_name
    admguy@guihost:~$ sudo chown admguy /custom_directory/custom_name
    admguy@guihost:~$ export XAUTHORITY=/custom_directory/custom_name
    admguy@guihost:~$ xclock

This procedure would have the same effect as before, but it is not recommended.

<br>

Any other GUI program can be run on `targetuser`'s display as a process owned by `admguy`: for example, `xcalc`. To display a message, [X Window System][x-window-system-wiki] provides:

    admguy@guihost:~$ xmessage -buttons Yes,No,Maybe -nearmouse "Do you enjoy pineapple on pizza?"

Buttons names are customizable, as well as their exit values, which can be displayed through `$?` once the window has been closed by `targetuser` through some of the buttons. Default exit values are:

- `101` if button `Yes` is pressed;
- `102` if button `No` is pressed;
- `103` if button `Maybe` is pressed;
- `1` if window is closed through upper-right `x`.

See [`man xmessage`][xmessage-man] for more information. Please, be careful that character encodings other than ASCII may not be correctly displayed or may need dedicated settings.

<br>

Sysadm can also act as `targetuser` through `sudo`. This way, he does not need to acquire `.Xauthority` information into his own `.Xauthority` file, because he will represent `targetuser` to all intents and purposes and will have a direct access to `~targetuser/.Xauthority`. Sysadm needs only to reproduce the correct environment `targetuser` would have: this implies, in this case, just to set the `DISPLAY` variable.

    admguy@guihost:~$ sudo -u targetuser DISPLAY=:0 xmessage -buttons Yes,No,Maybe -nearmouse "Do you enjoy pineapple on pizza?"

Note that `DISPLAY` can be here set just for the scope of the line. This has been tested on Ubuntu 16.04, 18.04, KDE Neon 5.14. This way, process is owned not by sysadm, but by `targetuser`:

    admguy@guihost:~$ ps -aux | grep xmessage
    targetuser   24928  0.0  0.0  50548  5604 pts/8    S+   18:01   0:00 xmessage -buttons Yes,No,Maybe -nearmouse "Do you enjoy pineapple on pizza?"

Alternatively, quick notifications can be sent:

    admguy@guihost:~$ sudo -u targetuser DISPLAY=:0 notify-send -u critical "Order pineapple pizza"

While `xmessage` is made available by [X Window System][x-window-system-wiki], `notify-send` availability depends on the Desktop Environment. [GNOME][gnome-home], [Xfce][xfce-home] and [KDE Neon][kde-neon-home] offer it, but other may not. Urgency level `critical` lets the notification never disappear; choosing other levels may cause it to be automatically closed after some time. See [`man nofity-send`][notify-send-man] for more information.

This has been tested on Ubuntu 16.04. As regards Ubuntu 18.04 and KDE Neon 5.14, also the environment variable `XDG_RUNTIME_DIR` must be specified, which is usually `/run/user/<user_ID>`.

In such cases, if possible, run

    targetuser@guihost:~$ notify-send -u critical "Order pineapple pizza"

for test purposes from a `targetuser` GUI terminal emulator. If `targetuser` is able to correctly display the message, while `admguy` with `sudo -u targetuser` is apparently not, as anticipated the only difference may be represented by the environment variable values in these two contexts. 

Run `env` in the `targetuser` GUI terminal emulator to get a full list of all the environment variables; then, compare their names and their values to the ones in the shell obtained, from `admguy` prompt, after typing `sudo -u targetuser bash`, assuming [`bash`][bash-home] as shell. If a variable `VARIABLE` with value `value` is defined in the first environment but not in the second one, type `export VARIABLE=name` in the `sudo -u targetuser bash` shell. Run `notify-send -u critical "Order pineapple pizza"` again. If it fails, try to `export` another `VARIABLE`, `value` pair. Not all the variables in `targetuser`'s environment are always needed.

While for Ubuntu 16.04 variable `DISPLAY` was sufficient, Ubuntu 18.04 and KDE Neon 5.14 required also `XDG_RUNTIME_DIR` in order to provide the same result.

User ID can be obtained through `getent passwd`. Assume that `targetuser`'s user ID is `1000`:

    admguy@guihost:~$ sudo -u targetuser bash
    admguy@guihost:~$ export DISPLAY=:0
    admguy@guihost:~$ export XDG_RUNTIME_DIR=/run/user/1000
    admguy@guihost:~$ notify-send -u critical "Order pineapple pizza"

This way, sysadm will be able through the graphical interface to poll `targetuser` with `xmessage`, or to simply notify him some messages with `notify-send`. Note that `xmessage` can be used to send notifications, too: moreover, it always lets sysadm to know if they have been read, because its process exits only when `targetuser` presses a button; a default button `okay` is included also when no `-buttons` are specified.



[gui-wiki]: https://en.wikipedia.org/wiki/Graphical_user_interface
[x-window-system-wiki]: https://en.wikipedia.org/wiki/X_Window_System
[linux-wiki]: https://en.wikipedia.org/wiki/Linux
[ssh-wiki]: https://en.wikipedia.org/wiki/Secure_Shell
[env-wiki]: https://en.wikipedia.org/wiki/Env
[unix-socket-wiki]: https://en.wikipedia.org/wiki/Unix_domain_socket
[xauth-man]: https://www.x.org/releases/X11R7.7/doc/man/man1/xauth.1.xhtml
[xmessage-man]: https://www.x.org/archive/X11R7.6/doc/man/man1/xmessage.1.xhtml
[gnome-home]: https://www.gnome.org/
[xfce-home]: https://www.xfce.org/
[kde-neon-home]: https://neon.kde.org/
[notify-send-man]: http://manpages.ubuntu.com/manpages/bionic/man1/notify-send.1.html
[bash-home]: https://www.gnu.org/software/bash/
