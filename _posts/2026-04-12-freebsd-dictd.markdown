---
layout: post
title:  "dictd in FreeBSD"
date:   2026-04-12 22:39:00 +0200
categories: freebsd
---
**Disclaimer**: this is not an authoritative guide; it is just my understanding of this topic.

### Server setup

The Dictionary Server Protocol (DICT) allows to easily provide (as a server) and query (as a client) dictionaries across networks. It relies upon TCP, with plain text, even human readable messages.

In FreeBSD package repository, the DICT server is provided by [`net/dictd`][package]. Its manpage is `dictd(8)`. The few and simple installed files are:

```
$ pkg info -l dictd
dictd-1.13.3:
        /usr/local/bin/dictzip
        /usr/local/etc/dictd.conf.sample
        /usr/local/etc/rc.d/dictd
        /usr/local/sbin/dictd
        /usr/local/share/licenses/dictd-1.13.3/GPLv2
        /usr/local/share/licenses/dictd-1.13.3/LICENSE
        /usr/local/share/licenses/dictd-1.13.3/catalog.mk
        /usr/local/share/man/man1/dictzip.1.gz
        /usr/local/share/man/man8/dictd.8.gz
```

Between them, a sample configuration file is provided, with a bunch of entries like:

```
database jargon    { data "/usr/local/lib/dict/jargon.dict.dz"
                     index "/usr/local/lib/dict/jargon.index" }
```

The `Database Section`, with its `database` entries, is the only mandatory one.

All the entries refer to the directory `/usr/local/lib/dict/`, which does not exist, it is not created by the package installation, and it is of course not populated by the `.dict.dz` and `.index` files mentioned in this example configuration file. Is there any FreeBSD dictionary package which automatically does this?

Some suitable dictionaries can be fetched from [https://freedict.org/](https://freedict.org/), in the Downloads section. Choose "Dictd: Computers", "Dictionary Downloads", then pick a language, for example "English". Here, the "English - Catalan, version 2025.11.23 with 35163 headwords" will be used as an example.

Fetch the archive file `freedict-eng-cat-2025.11.23.dictd.tar.xz`, verify its checksum and extract the files.

As stated in the `FILES` section of the manpage,

```
      /usr/local/share
              The default directory for dictd databases (.index and .dict[.dz]
              files)
```

This is a different directory with respect to the one used in `/usr/local/etc/dictd.conf.sample`.

The databases may then be stored in a subdirectory named `dict` or `dictd`. In the `SEE ALSO` section, some `examples/dictd*.conf` appear, but nothing similar is installed or present in the system, except for the already considered `/usr/local/etc/dictd.conf.sample`, whose directory is however not named `examples`. The `webster(1)` manpage mentioned in the same section is not present, as well: maybe this information is referred to Linux or some other OS and the manpage for FreeBSD has not been updated.

According to the lines in `/usr/local/etc/dictd.conf.sample`, two files are needed for each dictionary in this format: the `.dict.dz` and `.index` files. Both are available in `freedict-eng-cat-2025.11.23.dictd.tar`. After creating the directory `/usr/local/share/dictd`, `eng-cat.dict.dz` are `eng-cat.index` are copied there.

First, install the packages `net/dictd` and `textproc/dict`, respectively the DICT server and client. Then, add

```
dictd_enable="YES"
```

in `/etc/rc.conf`.

The `dictd.conf.sample` configuration file must be completely rewritten and the chosen directory must be populated with dictionary files. As stated in the manpage, the configuration file is `/usr/local/etc/dictd.conf`.

It is tricky to verify that the server is actually running and that the information included in the configuration file is correct. In fact, if for example some non-existing paths are present in `/usr/local/etc/dictd.conf`, the `dictd` daemon offers no clue about the error and even about its running status.

```
# service dictd start
Starting dictd.
# service dictd stop
dictd not running?
# service dictd status
dictd is not running.
# ps -aux | grep dic
root   61402   0.0  0.1   13836  2132  2  S+   00:19       0:00.00 grep dic
```

No process has been created for `dictd`, but on the other hand no error followed `Starting dictd.`. Nothing changes running directly the daemon executable with the `-v` option: `/usr/local/sbin/dictd -v`.

Checking the path provided by `dictd(8)`:

```
$ cat /var/run/dictd.pid
44217
```

The file has been modified exactly when `service dictd start` has been performed. But

```
# ps -aux | grep dictd
# ps -aux | grep 44217
```

show nothing.

```
# sockstat -4 -l | grep 2628
```

has no output. `2628` is the default port number (not modified in the configuration file) for the DICT protocol.

There is also no information regarding `dictd` in `/var/log`.

After fixing the configuration file, cleaning all the non-existing entries, and leaving only the existing one,

```
database eng-cat    { data "/usr/local/share/dictd/eng-cat.dict.dz"
                     index "/usr/local/share/dictd/eng-cat.index" }
```

the server runs and a client can connect to it.

```
# service dictd start
Starting dictd.
# service dictd status
dictd is running as pid 78213.
# sockstat -4 -l
USER     COMMAND    PID   FD  PROTO  LOCAL ADDRESS         FOREIGN ADDRESS
nobody   dictd      78213 3   tcp4   *:2628                *:*
...
# ps -aux | grep dictd
nobody 78213   0.0  0.2   18284  7284  -  Is   22:47       0:00.01 /usr/local/sbin/dictd
```

Note that only a privileged user can list `dictd` between the running process; a regular user can not. From a client:

```
$ dict -h <dictd_host_IP> -d eng-cat wood
1 definition found

From English-català FreeDict+WikDict dictionary ver. 2025.11.23 [eng-cat]:

  wood //wʊd// <n>
  1. fusta 2.
  (uncountable) The substance making up the central part of the trunk and branches of a tree. Used as a material for construction, to manufacture various items, etc. or as fuel.
   3.
  (countable) The wood from a particular species of tree.
   4.
  (countable, golf) A type of golf club, the head of which was traditionally made of wood.
  2. llenya
  Firewood.
  3. bosc
  (music) A woodwind instrument.
```

If a database is added or modified, it's possible to make `dictd` aware without restarting the service. From `dictd(8)`:

```
       SIGHUP causes dictd to reread configuration file and reinitialize
       databases.
```

So, send `SIGHUP` with `kill(1)` to the `dictd` process. Using the `pid` listed above:

```
kill -s HUP 78213
```

This will not end the process.


### The first message of the server

The DICT protocol is defined in [RFC2229][rfc1].

The initial server message is a banner and it is worth inspecting it. The RFC states:

```
   When a client initially connects to a DICT server, a code 220 is sent
   if the client's IP is allowed to connect:

             220 text capabilities msg-id

   The code 220 is a banner, usually containing host name and DICT
   server version information.

   The second-to-last sequence of characters in the banner is the
   optional capabilities string, which will allow servers to declare
   support for extensions to the DICT protocol.  The capabilities string
   is defined below:

             capabilities =  ["<" msg-atom *("." msg-atom) ">"]
             msg-atom     =  1*<any CHAR except SPACE, CTLs,
                                "<", ">", ".", and "\">

   Individual capabilities are described by a single msg-atom.  For
   example, the string <html.gzip> might be used to describe a server
   that supports extensions which allow HTML or compressed output.
   Capability names beginning with "x" or "X" are reserved for
   experimental extensions, and SHOULD NOT be defined in any future DICT
   protocol specification.  Some of these capabilities may inform the
   client that certain functionality is available or can be requested.
   The following capabilities are currently defined:

       mime        The OPTION MIME command is supported
       auth        The AUTH command is supported
       kerberos_v4 The SASL Kerberos version 4 mechanism is supported
       gssapi      The SASL GSSAPI [RFC2078] mechanism is supported
       skey        The SASL S/Key [RFC1760] mechanism is supported
       external    The SASL external mechanism is supported

   The last sequence of characters in the banner is a msg-id, similar to
   the format specified in [RFC822].  The simplified description is
   given below:

       msg-id       =  "<" spec ">"            ; Unique message id
       spec         =  local-part "@" domain
       local-part   =  msg-atom *("." msg-atom)
       domain       =  msg-atom *("." msg-atom)
```

The client executable `dict` (available on FreeBSD as well as on Linux) may not expose the `220` message, but the server can be queried with just `nc`. So,

```
$ nc <dictd_host_IP> 2628
220 dictd_hostname dictd 1.13.3/rf on FreeBSD 14.3-RELEASE-p3 <auth.mime> <3.56515.1775808671@dictd_hostname>
```

Other than the `dictd` version, the OS running in the machine and its exact version are being revealed. This would be golden information for any potential attacker. However, it is a feature of **this** implementation of `dictd`: the RFC does not consider such information as mandatory. As shown above, it is just vaguely stated that the `220` message «usually» contains «host name and DICT server version information». The OS is not even considered.

The `dictd` provided by `pkg` is the one in [Sourceforge][sourceforge] and its GitHub repository should be [https://github.com/cheusov/dictd](https://github.com/cheusov/dictd). This may be the _only_, or at least the _main_, `dictd` version available for the Unix-like OSs.

According to `dictd(8)`, in the `Global Settings Specification` section of the configuration file `dictd.conf`, some parameters may be specified to restrict the information shown by the server:

```
              site_no_banner boolean
                     By default SHOW SERVER command outputs information about
                     dictd version and an operating system type.  This option
                     disables this.

              site_no_uptime boolean
                     By default SHOW SERVER command outputs information about
                     uptime of dictd , a number of forks since startup and
                     forks per hour.  This option disables this.

              site_no_dblist boolean
                     By default SHOW SERVER command outputs internal
                     information about databases, such as a number of
                     headwords, index size and so on.  This option disables
                     this.
```

In `dictd.conf`:

```
global {
        site_no_banner true
        site_no_uptime true
        site_no_dblist true
}
```

However, this avoids the server from providing such information only in the reply to the client `SHOW SERVER` command:

```
$ nc <dictd_host_IP> 2628
220 dictd_hostname dictd 1.13.3/rf on FreeBSD 14.3-RELEASE-p3 <auth.mime> <20.95463.1775826797@dictd_hostname>
SHOW SERVER
114 server information
.
250 ok
QUIT
221 bye [d/m/c = 0/0/0; 9.000r 0.000u 0.000s]
```

The initial `220` banner has not changed.


### Inspecting the source code

In order to definitively determine if the OS version can be hidden, the source code must be checked out.

In the above GitHub or SourceForge repository, the relevant files are `dictd.c` and `daemon.c`. Part of the `220` banner is generated in `dictd.c`:

```
const char *dict_get_banner( int shortFlag )
{
   static char    *shortBuffer = NULL;
   static char    *longBuffer = NULL;
   struct utsname uts;
  
   if (shortFlag && shortBuffer) return shortBuffer;
   if (!shortFlag && longBuffer) return longBuffer;
  
   uname( &uts );

   shortBuffer = xmalloc(256);
   snprintf(
      shortBuffer, 256,
      "%s %s", err_program_name(), id_string () );

   longBuffer = xmalloc(256);
   snprintf(
      longBuffer, 256,
      "%s %s/rf on %s %s", err_program_name(), id_string (),
      uts.sysname,
      uts.release );

   if (shortFlag)
      return shortBuffer;

   return longBuffer;
}
```

Note that the strings pointers are declared as `static`:

```
   static char    *shortBuffer = NULL;
   static char    *longBuffer = NULL;
```

The first time `dict_get_banner()` is called, these two lines are actually executed and `shortBuffer` and `longBuffer` are `NULL` pointers in the beginning. The subsequent times `dict_get_banner()` is called, instead, due to the `static` statement, the above two lines will be ignored: `shortBuffer` and/or `longBuffer` _will keep their value_ (assumed at the end of the first `dict_get_banner()` call), even after the function ended. They are _not_ overwritten.

```
   if (shortFlag && shortBuffer) return shortBuffer;
   if (!shortFlag && longBuffer) return longBuffer;
```

are used for subsequent calls of `dict_get_banner`: if there's a `shortFlag`, and `shortBuffer` has already been created, simply return the buffer string and do not execute the rest of the code; if there's no `shortFlag` and `longBuffer` has already been created, simply return it and do not execute the rest of the function code.

The first time `dict_get_banner` is called, `longBuffer` is filled this way:

```
   snprintf(
      longBuffer, 256,
      "%s %s/rf on %s %s", err_program_name(), id_string (),
      uts.sysname,
      uts.release );
```

Four strings are printed into `longBuffer`:
- the output of `err_program_name()`;
- `id_string()` (why does it have a space between `id_string` and `()` in the original code? This is maybe a typo);
- `uts.sysname`;
- `uts.release`.

`id_string()` is defined just above `dict_get_banner()` and it prints the macro `DICT_VERSION` on a string. `DICT_VERSION` is here `1.13.3`, the `dictd` version:

```
static const char *id_string (void)
{
   static char buffer [BUFFERSIZE];

   snprintf( buffer, BUFFERSIZE, "%s", DICT_VERSION );

   return buffer;
}
```

Note the position of the verbatim string `/rf on` in

```
220 dictd_hostname dictd 1.13.3/rf on FreeBSD 14.3-RELEASE-p3 <auth.mime> <20.95463.1775826797@dictd_hostname>
```

The first two strings printed into `longBuffer` therefore produce:

```
dictd 1.13.3
```

The last two strings produce

```
FreeBSD 14.3-RELEASE-p3
```

`uts.sysname` and `uts.release` are members of the `struct utsname uts`. The `struct utsname` is defined in `<sys/utsname.h>`, which is an `#include` of `dict.h`, and it is populated by `uname(3)` (which in fact is called before `snprintf`). On FreeBSD, in `/usr/include/sys/utsname.h`,

```
struct utsname {
        char    sysname[SYS_NMLN];      /* Name of this OS. */
        char    nodename[SYS_NMLN];     /* Name of this network node. */
        char    release[SYS_NMLN];      /* Release level. */
        char    version[SYS_NMLN];      /* Version level. */
        char    machine[SYS_NMLN];      /* Hardware type. */
}
```

To show them, use this simple test code compiled into the executable `uts_test`:

```
#include <stdio.h>
#include <sys/utsname.h>

int main()
{
        struct utsname uts;

        if (!uname(&uts)) {
                printf("sysname: %s\n", uts.sysname);
                printf("nodename: %s\n", uts.nodename);
                printf("release: %s\n", uts.release);
                printf("version: %s\n", uts.version);
                printf("machine: %s\n", uts.machine);
        }
        return 0;
}
```

The result is:

```
$ ./uts_test
sysname: FreeBSD
nodename: dictd_hostname
release: 14.3-RELEASE-p3
version: FreeBSD 14.3-RELEASE-p3 GENERIC
machine: amd64
```

On a Linux host,

```
$ ./uts_test
sysname: Linux
nodename: another_hostname
release: 6.17.0-20-generic
version: #20~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Thu Mar 19 01:28:37 UTC 2
machine: x86_64
```

The `nodename` here is not used to print the hostname in the `220` banner. The reason will be guessed below.

It can be observed that this `"%s %s/rf on %s %s"` string is just a part of the banner. It is:

```
dictd 1.13.3/rf on FreeBSD 14.3-RELEASE-p3
```

The `220` banner as a whole is actually formed in `daemon.c`. The relevant function is:

```
static void daemon_banner( void )
{
   time_t         t;

   time(&t);

   snprintf( daemonStamp, sizeof (daemonStamp), "<%d.%d.%lu@%s>",
            _dict_forks,
            (int) getpid(),
            (long unsigned)t,
             net_hostname() );
   daemon_printf( "%d %s %s <auth.mime> %s\n",
                  CODE_HELLO,
                  net_hostname(),
                  dict_get_banner(0),
                  daemonStamp );
}
```

`snprintf` forms the last part of the banner, the `msg-id`: `"<%d.%d.%lu@%s>"` is the `msg-id` specified by the RFC, with 3 fields separated by dots and followed by `@dictd_hostname`.

`(int) getpid()`, second field of `msg-id`, is created through the `getpid()` syscall. It is the process ID of the forked process, which is the PID of the current process, created when the client requested a connection.

Each time a client creates a connection to the server, `dictd` forks: a new `dictd` process to serve that particular client is created. For example, before a test connection only a `dictd` process is active in the server:

```
# ps -aux | grep dict
nobody 68550   0.0  0.2   18284  6872  -  Ss   01:30       0:00.02 /usr/local/sbin/dictd
```

When the test connection has been launched, there is a fork:

```
#  ps -aux | grep dict
nobody 68550   0.0  0.2   18284  6872  -  Ss   01:30       0:00.02 /usr/local/sbin/dictd
nobody 78835   0.0  0.2   18284  7056  -  S    10:50       0:00.00 /usr/local/sbin/dictd
```

and the client has received the following message:

```
220 dictd_hostname dictd 1.13.3/rf on FreeBSD 14.3-RELEASE-p3 <auth.mime> <17.78835.1775811031@dictd_hostname>
```

Note that the second field `78835` of the `msg-id` `<17.78835.1775811031@dictd_hostname>` is exactly the PID of the forked `dictd` process. The fact that it increases from one connection to the subsequent ones is just related to the OS (and it may not be always the case: some OSs randomize PIDs).

The first field of the `msg-id` is `_dict_forks`, the number of forks (so, the number of client connections) occurred since when the service `dictd` was started. The `int _dict_forks` variable is incremented in `dictd.c` just before `fork()` is called:

```
static int start_daemon( void )
{
   pid_t pid;     

   ++_dict_forks; 
   switch ((pid = fork())) {...}
...
}
```

and in turn `start_daemon()` is called inside the main `for (;;) {...}` loop in `dictd.c`, which will be shown below. The number of forks, together with the service `dictd` uptime, is explicitly shown as response to the DICT `SHOW SERVER` command, if `site_no_uptime true` is not present in `dictd.conf`.

`time(&t)` stores in variable `t` the seconds since the EPOCH representing the time when the server was contacted by the client and the process forked. It is the third field of `msg-id` after `(long unsigned)` casting, printed in the string `daemonStamp` as `%lu` (`unsigned long`).

The `@` is printed verbatim. As already noted, the hostname is not taken from the `uts` struct, but instead through the function `net_hostname()` defined in the sourcefile `net.c`, which uses the `gethostname(2)` syscall and performs some more checks, which are not performed by a simple call to `uname(3)`, so this may be the reason why the hostname is obtained this way and not through the `uts` struct populated by `uname(3)`.

```
const char *net_hostname( void )
{
   static char hostname[128] = "";
   int err;

   if (!hostname[0]) {
      if (err = gethostname(hostname, sizeof(hostname)), err != 0) {
         fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(err));
         exit(EXIT_FAILURE);
      }
   }

   hostname[sizeof(hostname)-1] = '\0';
   return hostname;
}
```

The whole banner is then produced:

```
daemon_printf( "%d %s %s <auth.mime> %s\n",
                  CODE_HELLO,
                  net_hostname(),
                  dict_get_banner(0),
                  daemonStamp );
```

`%d %s %s <auth.mime> %s\n` is what the client received.

The first integer `%d` is the macro `CODE_HELLO`, defined in `codes.h` as:

```
#define CODE_HELLO                   220 /* text msg-id */
```

It follows the hostname, again obtained through the custom function `net_hostname()` instead of from the `uts` struct. Then, it follows the output of `dict_get_banner(0)` from `dictd.c`, called with **no `shortFlag`** (the `0` argument).

There is no function trying to determine the _capabilities_ (as they are defined in the RFC) of the current setup: `auth` and `mime` are assumed to be the only ones always available, so simply `<auth.mime>` is printed verbatim.

The last `%s` is the `daemonStamp` string `"<%d.%d.%lu@%s>"`, as produced by the `snprintf` shown above.

The function `daemon_banner()`, which includes `daemon_printf()`, is called at some point by the function devoted to handle the connection with the client:

```
int _handleconn (int error) { ... }
```

in `daemon.c`.

`dict_get_banner()` has two possible outcomes: `shortBuffer` and `longBuffer`, according to the initial checks. In the `shortBuffer` version, no OS information is printed. The choice is only determined by the value `int shortFlag` passed to the function, which may be `0` (`shortFlag` not active) or `1` (`shortFlag` active).

`dict_get_banner()` is called only as `dict_get_banner(0)` inside `daemon.c`, with *no options* modifying this behaviour. Inside `dictd.c`, there's only one `dict_get_banner(1)` call:

```
   for (;;) {
      dict_setproctitle( "%s: %d/%d",
                         dict_get_banner(1),
                         _dict_forks - _dict_reaps,
                         _dict_forks );
   ...
```

However, `dict_setproctitle` may just act on the process name, modifying `argv[0]`, so this is not about the lines printed by the server to the client, but just about how the OS sees the `dictd` process.

In conclusion, with this 1.13.3 code version, when generating the lines to be printed by the `dictd` server to the client in the `200` banner, only the `dict_get_banner(0)` version is used. `dict_get_banner(0)` is the function call performed by `daemon_banner()`, the only time it is called by the handler `_handleconn` of the connection with the client: **the OS name and version are always printed**, with **no options that can avoid them**.


As a side note, as partially anticipated, the main loop in `dictd.c`, which makes the server listening for incoming connections, is inside its `int main (...)` function. It is a conditionless `for`:

```
   for (;;) {
      dict_setproctitle( "%s: %d/%d",
                         dict_get_banner(1),
                         _dict_forks - _dict_reaps,
                         _dict_forks );

      if (flg_test(LOG_SERVER))
         log_info( ":I: %d accepting on %s\n", getpid(), daemon_service );
      ...
```

Being the file with `int main`, `dictd.c` is the most important sourcefile; `daemon.c` is where the functions of the server are implemented.

In `daemon.c`,

```
static void daemon_show_server (...) {
   ...
   if (!site_info_no_banner){
      daemon_printf( "%s\n", dict_get_banner(0) );
   }
   ...
}
```

This is about the DICT procotol command `SHOW SERVER`. `site_info_no_banner` represents the value of the option `site_no_banner` in `dictd.conf` as shown above: if the option is active, do not show the `dictd` version and OS name and version while replying to `SHOW SERVER`.

The same as regards the presence of the lines `site_no_uptime true` and `site_no_dblist true` in `dictd.conf`:

```
   if (!site_info_no_uptime && !inetd){ ...}
   if (!site_info_no_dblist && count_databases()) { ... }
```

There are several public `dictd` servers, using this `dictd` or even older versions, and they all expose their running OSs.


### Example raw communication

The RFC 2229 defines several text commands. Between them:

```
3.2.  The DEFINE Command

   DEFINE database word

3.2.1.  Description

   This command will look up the specified word in the specified
   database.  All DICT servers MUST implement this command.
...
3.9.  The QUIT Command

   QUIT

3.9.1.  Description

   This command is used by the client to cleanly exit the server.  All
   DICT servers MUST implement this command.

3.9.2.  Responses

             221 Closing Connection

   This response code requires no special parameters.
```

For example:

```
$ nc dictd_hostname 2628
220 dictd_hostname dictd 1.13.3/rf on FreeBSD 14.3-RELEASE-p3 <auth.mime> <15.49358.1776086235@dictd_hostname>
DEFINE eng-cat wood
150 1 definitions retrieved
151 "wood" eng-cat "English-català FreeDict+WikDict dictionary ver. 2025.11.23"
wood //wʊd// <n>
1. fusta 2.
(uncountable) The substance making up the central part of the trunk and branches of a tree. Used as a material for construction, to manufacture various items, etc. or as fuel.
 3.
(countable) The wood from a particular species of tree.
 4.
(countable, golf) A type of golf club, the head of which was traditionally made of wood.
2. llenya
Firewood.
3. bosc
(music) A woodwind instrument.
.
250 ok [d/m/c = 1/0/13; 0.000r 0.000u 0.000s]
SHOW SERVER
114 server information
.
250 ok
QUIT
221 bye [d/m/c = 0/0/0; 8.000r 0.000u 0.000s]
```

This was a server with `site_no_banner`, `site_no_uptime`, `site_no_dblist` set to `true`.





[package]: https://www.freshports.org/net/dictd
[rfc1]: https://datatracker.ietf.org/doc/html/rfc2229
[sourceforge]: https://sourceforge.net/projects/dict/
