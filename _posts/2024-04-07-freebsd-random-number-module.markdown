---
layout: post
title:  "Example FreeBSD module with a random number generator output"
date:   2024-04-08 20:24:00 +0200
categories: freebsd, kernel, modules
---
**Disclaimer**: this is not an authoritative guide; it is just my understanding of this topic.

The following code is built on:

```
$ uname -a
FreeBSD hostname 14.0-RELEASE FreeBSD 14.0-RELEASE #0 releng/14.0-n265380-f9716eee8ab4: Fri Nov 10 05:57:23 UTC 2023     root@releng1.nyi.freebsd.org:/usr/obj/usr/src/amd64.amd64/sys/GENERIC amd64
```

The [FreeBSD Architecture Handbook][arch-handbook] shows an [example module][char-device] which creates a character device in the `/dev` directory, corresponding to a virtual device which performs some basic operations: the `root` user can write a string to the device, and the same string can be read back by reading from the device.

A very similar procedure (even if more outdated) is presented in Chapter 1 of _FreeBSD device drivers, A guide for the intrepid_ by Joseph Kong, _No Starch Press_, 2012. To make the example simpler, it explicitly avoids the usage of `uiomove` to copy data from user space to kernel space, but it also states that `uiomove` is _the proper way_ to perform such an operation. Here, `uiomove` is used (for the opposite case: to copy data from kernel space to user space), as in the Architecture Handbook [example][char-device].

As an exercise, define a kernel module which creates a character device to simulate a die roll. The device can not be written, but only read: after a single read operation, it returns only a single byte, representing the numerical value obtained by a die roll. Each different read will provide the result of a different die roll. The number of faces of the die is stored in the variable `d_size`. For the sake of simplicity the character device returns just a single byte, so `d_size` can not exceed the value `9`.

Use a suitable random number generator mechanism available inside the kernel to obtain such a value. The kernel programming interface is documented in Section 9 of the manpages. To look for a specific topic in a specific section, a couple of solutions can be chosen:

<!-- https://stackoverflow.com/a/18089124 https://www.w3schools.io/file/markdown-comments/ -->

1. Trivially,

    ```
    $ cd /usr/share/man/man9
    $ ls -l | grep -i random
    -r--r--r--    7 root wheel  2227 Nov 10 09:32 arc4random.9.gz
    -r--r--r--    7 root wheel  2227 Nov 10 09:32 is_random_seeded.9.gz
    -r--r--r--    7 root wheel  2227 Nov 10 09:32 random.9.gz
    -r--r--r--    4 root wheel  1598 Nov 10 09:32 random_harvest.9.gz
    -r--r--r--    4 root wheel  1598 Nov 10 09:32 random_harvest_direct.9.gz
    -r--r--r--    4 root wheel  1598 Nov 10 09:32 random_harvest_fast.9.gz
    -r--r--r--    4 root wheel  1598 Nov 10 09:32 random_harvest_queue.9.gz
    -r--r--r--    7 root wheel  2227 Nov 10 09:32 read_random.9.gz
    -r--r--r--    7 root wheel  2227 Nov 10 09:32 read_random_uio.9.gz
    -r--r--r--    7 root wheel  2227 Nov 10 09:32 srandom.9.gz
    ```

2. Using `apropos(1)':

    ```
    $ apropos -s 9 random
    prng, prng32, prng32_bounded, prng64, prng64_bounded(9) - Kernel pseudo-random number generators
    random, arc4rand, arc4random, is_random_seeded, read_random, read_random_uio, srandom, arc4random_buf(9) - supply pseudo-random numbers
    random_harvest, random_harvest_direct, random_harvest_fast, random_harvest_queue(9) - gather entropy from the kernel for the entropy device
    ```

The manpage `random(9)` suggests the `arc4random()` function, which returns `very good quality random numbers, suited for security-related purposes``.


### First version

The module is created with the same structure suggested in the [Architecture Handbook][char-device]. First of all, some `#include`s are necessary: their purpose is shown in the code comments.

```
#include <sys/systm.h>          /* uprintf */
#include <sys/param.h>          /* defines used in kernel.h */
#include <sys/module.h>         /* necessary to create the kernel module itself */
#include <sys/kernel.h>         /* types used in module initialization */
#include <sys/conf.h>           /* cdevsw struct */
#include <sys/uio.h>            /* uio struct */
#include <sys/malloc.h>		/* malloc() will be used within this module */
#include <sys/libkern.h>        /* arc4random(). See random(9) */
```

There is not a single way to proceed. While maybe not strictly necessary, sticking to the [Architecture Handbook example][char-device] let's create anyway a buffer:

```
#define BUFFERSIZE 2
```

Then,

```
/* Function prototypes */
static d_open_t      rolld_open;
static d_close_t     rolld_close;
static d_read_t      rolld_read;

/* Character device entry points */
static struct cdevsw rolld_cdevsw = {
        .d_version = D_VERSION,
        .d_open = rolld_open,
        .d_close = rolld_close,
        .d_read = rolld_read,
        .d_name = "rolld",
};
```

Note that there is no `rolld_write` function, because it is not needed here, as previously mentioned.

```
struct s_roll {
        char msg[BUFFERSIZE + 1];
        int len;
};

/* vars */
static struct cdev *rolld_dev;
// it must be <= 9. See rolld_read
static uint32_t d_size = 6;
static struct s_roll *rolldmsg;
```

The buffer needs a `malloc()` operation: part of memory will be devoted to it, as `vmstat -m` can show while the module is used, providing as description the second argument of `MALLOC_DEFINE`.

```
MALLOC_DECLARE(M_ROLLDBUF);
MALLOC_DEFINE(M_ROLLDBUF, "rolldbuffer", "minimal buffer for rolld module");
```

Then, define the operations to be performed when the module is loaded into the kernel, to create the device:

```
static int
rolld_loader(struct module *m __unused, int what, void *arg __unused)
{
        int error = 0;

        switch (what) {
        case MOD_LOAD:
                error = make_dev_p(MAKEDEV_CHECKNAME | MAKEDEV_WAITOK,
                    &rolld_dev,
                    &rolld_cdevsw,
                    0,
                    UID_ROOT,
                    GID_WHEEL,
                    0444,               /* write is not implemented, so no need to write to this device */
                    "rolld");
                if (error != 0)
                        break;

                rolldmsg = malloc(sizeof(*rolldmsg), M_ROLLDBUF, M_WAITOK |
                    M_ZERO);
```

`rolldmsg` has been defined as `static struct s_roll *rolldmsg`, so it is a pointer to a `struct s_roll`: after having allocated a region of memory of the size of such a `struct`, `malloc` returns a pointer to that region, which is then stored in `rolldmsg`. The region will be filled during the module operation (in particular, here, within the `rolld_read` function).

Note that the device `/dev/rolld` will have permissions `r--r--r--`, being read-only.

Functions `rolld_open` and `rolld_close` are left, but without printing anything to the user: this way, they are probably useless.

The function `rolld_read` is the core of the module:

```
static int
rolld_read(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
        size_t amt;
        rolldmsg->len = 2;
        char random_item;
        int error;

```

As established by `rolldmsg->len = 2`, the buffer here will just have two characters: the digit representing the result of the die roll and a newline character, to make the ouput more readable.

```
        random_item = (char) arc4random();
```

`arc4random()` returns a `uint32_t` value. It must be represented by a character here: first of all, cast it to an `int`-like type. Then:

```
        rolldmsg->msg[0] = 49 + (random_item % d_size);
```

Fill the first element of the buffer with the ASCII character corresponding to the chosen number. With `d_size = 6`, `random_item` may be `0`, `1`, `2`, `3`, `4`, `5`: so, start from ASCII decimal value 49, corresponding to `1`. When `random_item` is `0`, `1` is stored in `msg[0]`.

Now, fill the second element of the buffer with a newline and terminate the buffer string with a NULL character:

```
        rolldmsg->msg[1] = '\n';
        rolldmsg->msg[rolldmsg->len] = 0;
```

The contents of the buffer must be transferred to the user through `uiomove`. First, define `amt` as:

```
       amt = MIN(uio->uio_resid, uio->uio_offset >= rolldmsg->len + 1 ? 0 :
            rolldmsg->len + 1 - uio->uio_offset);
```

This check is essential. Then,

```
        if ((error = uiomove(rolldmsg->msg, amt, uio)) != 0)
                uprintf("uiomove failed!\n");

        return (error);
}
```

actually prints the digit resulting from the die roll to the user.

If instead `amt` is replaced with a fixed value, `cat /dev/rolld` continuously prints the buffer contents as if the die was rolled endless times: `uiomove(rolldmsg->msg, 1, uio)` prints endless numbers between 1 and 6; `uiomove(rolldmsg->msg, 2, uio)` does the same, but each number is in a newline (because newline is the second element of the buffer `msg`). Also, for some reason `tail -c 1 /dev/rolld` produces no output, as well as `tail -c 1 /dev/random`.

Finally,

```
DEV_MODULE(rolld, rolld_loader, NULL);
```

To build the module, the same `Makefile` structure specified in the [Architecture Handbook][makefile-structure] can be used:

```
KMOD=rolld
SRCS=rolld.c

.include <bsd.kmod.mk>
```

`rolld.c` is the name of the kernel module sourcefile. Before running `make`, be sure that `/usr/src` (or any equivalent directory) is populated by the kernel sourcecode, so that the `#include` files will be available.

```
$ make
Warning: Object directory not changed from original /path/to/this/example
cc  -O2 -pipe  -fno-strict-aliasing -Werror -D_KERNEL -DKLD_MODULE -nostdinc   -include /path/to/this/example/opt_global.h -I. -I/usr/src/sys -I/usr/src/sys/contrib/ck/include -fno-common  -fno-omit-frame-pointer -mno-omit-leaf-frame-pointer -fdebug-prefix-map=./machine=/usr/src/sys/amd64/include -fdebug-prefix-map=./x86=/usr/src/sys/x86/include -fdebug-prefix-map=./i386=/usr/src/sys/i386/include     -MD  -MF.depend.rolld.o -MTrolld.o -mcmodel=kernel -mno-red-zone -mno-mmx -mno-sse -msoft-float  -fno-asynchronous-unwind-tables -ffreestanding -fwrapv -fstack-protector -Wall -Wstrict-prototypes -Wmissing-prototypes -Wpointer-arith -Wcast-qual -Wundef -Wno-pointer-sign -D__printf__=__freebsd_kprintf__ -Wmissing-include-dirs -fdiagnostics-show-option -Wno-unknown-pragmas -Wno-error=tautological-compare -Wno-error=empty-body -Wno-error=parentheses-equality -Wno-error=unused-function -Wno-error=pointer-sign -Wno-error=shift-negative-value -Wno-address-of-packed-member -Wno-format-zero-length   -mno-aes -mno-avx  -std=gnu99 -c rolld.c -o rolld.o
ld -m elf_x86_64_fbsd -warn-common --build-id=sha1 -T /usr/src/sys/conf/ldscript.kmod.amd64 -r  -o rolld.ko rolld.o 
:> export_syms
awk -f /usr/src/sys/conf/kmod_syms.awk rolld.ko  export_syms | xargs -J% objcopy % rolld.ko
objcopy --strip-debug rolld.ko
```

Load the module from a root shell specifying its (relative or absolute) path:

```
# kldload -v ./rolld.ko
Loaded /path/to/this/example/rolld.ko, id=5
```

The module can now be shown by `kldstat(8)`:

```
# kldstat 
Id Refs Address                Size Name
 1   18 0xffffffff80200000  1d34598 kernel
 2    1 0xffffffff82800000   3e5cc8 zfs.ko
 3    1 0xffffffff82620000     3390 acpi_wmi.ko
 4    1 0xffffffff82624000     2a68 mac_ntpd.ko
 5    1 0xffffffff82627000     2218 rolld.ko
```

as well as the memory allocated for its internal buffer, labelled `rolldbuffer` as specified in `MALLOC_DEFINE`:

```
# vmstat -m                                                         
         Type InUse MemUse Requests  Size(s)                                                                                                                              
...
  rolldbuffer     1     1K        1  16
```

The device has been created and it is available to be used:

```
$ cd /dev
$ ls -l rolld 
cr--r--r--  1 root wheel 0x87 Apr  6 23:59 rolld
$ cat rolld 
6
$ cat rolld 
5
$ cat rolld 
5
$ cat rolld 
1
$ cat rolld 
3
$ cat rolld 
5
```

The full code for this module `rolld.c` and its `Makefile` can be found [here][rolld-repo].


### Second version

In the above section, given the Architecture Handbook [example code][char-device], the most unoriginal implementation for such a task has been presented.

A whole buffer `msg`, referenced by a related `struct s_roll`, is not even necessary in this case, when only a single 1 byte value must be provided to the user each time the file `/dev/rolld` is read.

Consider [this simpler code][rolld-repo-2] now: it is simpler in the sense that it does not create a buffer, it does not have `struct s_roll` and it does not need to use `malloc()'. Messages in `rolld_open`, `rolld_close` and `rolld_read` are instead produced through `uprintf` to better distinguish what code is being executed, as suggested [here][post-2]. Moreover, there is no translation between the raw `random_out` value and an ASCII printable digit.

This will allow to get more insight into the operations performed by the code.

The `rolld_read` function now is:

```
static int
rolld_read(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
        uprintf("Hello.\n");
        uprintf("uio_offset: %ld\n", uio->uio_offset);
        uprintf("uio_resid: %ld\n", uio->uio_resid);
        uint8_t random_out;

        random_out = arc4random() % d_size;

        if (uio->uio_offset > 0) {
                uprintf("I'm zero!\n");
                return 0;
        }
        uprintf("Returned value is: %d\n", (uiomove(&random_out, 1, uio)));
        return 0;
}
```

The values provided by `random_out` may be integers between `0` and `5`, which correspond to the following ASCII characters:

|Code	|Character		|
|-------|-----------------------|
|`0`	|`NUL`	`^@` or `\0`	|
|`1`	|`SOF`	`^A`		|
|`2`	|`STX`	`^B`		|
|`3`	|`ETX`	`^C`		|
|`4`	|`EOT`	`^D`		|
|`5`	|`ENQ`	`^E`		|

Being them non-printable, it is better to use `cat(1)` with `-v` option, to show them without the need of other tools like `hexdump`. When `cat -v` is called on `/dev/rolld`, as suggested in [this post][post-2], this output is produced:

```
$ cat -v /dev/rolld 
Opened device "rolld" successfully.
Hello.
uio_offset: 0
uio_resid: 4096
Returned value is: 0
Hello.
uio_offset: 1
uio_resid: 4096
I'm zero!
Closing device "rolld".
^C
```

In this case, `random_out` had value `3` and this is shown at the end of all the operations, as `^C`. Note that the `/dev/rolld` driver opens the file just one time at the beginning of the execution of `cat(1)` and it closes the file one time, only when `cat(1)` ends its execution.

When `cat(1)` reads a file, it calls `read(2)` (or an equivalent syscall) repeatedly. In the case of `/dev/rolld`, then `read(2)` causes the execution of the function inside the driver which performs the read operation inside the device: `rolld_read`.

`cat(1)` calls `read(2)`regardless of what kind of data is read from the file, as soon as _some data_ is read: in fact, even special characters like `^D` (`EOT`) are received, and printed here as they are, without preventing `cat(1)` from calling `read(2)` again, subsequently.

As specified [in this post][stack-1], `EOF` (End Of File) is not a character: it is the _absence of a character_, and it is a _condition_ rather than a data. It is created by the shell, for example, when `^D` is pressed, or by the device driver, when no more data is available. EOF _at the low level is a `0` return value from `read()`_: `read(2)` is called, it is successful because it returns `0`, but it prints **no data**. **This** is the EOF condition: only when it is met, the caller `cat(1)` will _stop_ calling `read(2)` again and it ends its execution, now knowing that the whole file has been consumed.

In order to interface the internal operation of the driver with the external requests from the userland, `uio` provides a `struct uio`, which is passed to `rolld_read` as second argument. One element of this `struct` is `uio_offset`: it is just _information available within the kernel module_, in this case within the character device. It is _arbitrary_ how to use it, but here the following constraint will be assumed: it is not desirable that `cat(1)`, called once by the user, repeatedly calls `read(2)` and so repeatedly prints different values of `random_out`. The `rolld` driver should instead provide, for each `cat /dev/rolld`, a single `random_out` value, and then provoke the end the `cat(1)` execution, as if the end of the file was reached after this value. This way, the file `/dev/rolld` will correspond to the contents of the variable `random_out`, which is a _single_ value.

To obtain another value, run `cat /dev/rolld` again, run again `arc4random() % d_size` and so generate new contents for the variable `random_out`, and read it again. `uio_offset` must be used accordingly, to create such a behaviour. Then, the above output represents the following operations:

1. `cat(1)` calls `read(2)` the first time: `rolld_read` runs with `uio_offset` `0` and some data is returned by uiomove. The amount of data that is returned is determined by the second argument of this `uiomove` call:

    ```
    uiomove(&random_out, 1, uio)
    ```

    It is, therefore, just `1` byte. It is in this case the value of `random_out`, as desired.

2. Having received data from this first `read(2)`, `cat(1)` calls `read(2)` a second time: `rolld_read` runs with `uio_offset` `1` and this time `uiomove` is not even executed, because of this check:

    ```
    if (uio->uio_offset > 0) {
            uprintf("I'm zero!\n");
            return 0;
    }
    ```

    Nothing returns to `cat(1)` this time. This _absence_ of data is interpreted by `cat(1)` as the EOF condition, because a `read(2)` has been run successfully, but without obtaining data. The file is assumed to be _ended_ and `cat(1)` ends its execution. Only the value of `random_out` returned during the first `read(2)` and `rolld_read` call is shown: `3`, which corresponds to ASCII character `^C` according to the above table.

The value of `uio_resid` is determined by `cat(1)`, but it is not relevant as regards the functioning of this driver.

According to the code, ASCII character `^C` should actually be shown after the first `uio_resid: 4096` and before `Returned value is: 0`, because `uiomove(&random_out, 1, uio)` is run before `uprintf("Returned value is: %d\n", (uiomove(&random_out, 1, uio)));`. However, for some reason, `cat(1)` dumps `^C` only in the very end.

Do not confuse `Returned value is: 0` with the data provided by `uiomove`: here, `0` is just the `int` returned by `uiomove` to signal that it ran succesfully providing the data byte `3`, which translates to `^C`. The second time, instead, `uiomove` does not run at all: only `rolld_read` runs and provides no data to the caller, but only a `0` return value.

The above check `if (uio->uio_offset > 0)` is **essential** as regards the driver operation. Without it, nothing inside the driver will create the EOF condition, allowing `rolld_read` to be repeatedly called from `cat(1)`. Each time, `rolld_read` will run `arc4random() % d_size`, so it will generate a new `random_out` value, and it will run `uiomove`, which will tranfer data: just 1 byte (for an actual file with longer contents, this would also be very inefficient, because for the next byte a brand new `rolld_read` will be needed, with an unnecessary overhead). The first time `rolld_read` is called, `uio_offset` is `0` and `uiomove` trasfers the actual contents of `random_out`; the second time `rolld_read` is called, `uio_offset` is `1`, but it is ignored, `arc4random() % d_size` is run again and `uiomove` is called again, transfering 1 more byte; this process repeats indefinitely, with `uio_offset` increasing by 1 each time and `cat(1)` never stopping calling `read(2)`.

This is the reason why the parameter `uio_offset` must be carefully handled inside the character device driver, to obtain the desired behaviour from it, towards the user.

As regards these observations, the suggestions in [this thread][thread-1], and in particular in [this post][post-1], have been of great help. It is also mentioned there that the usage of `if (uio->uio_offset > 0)` can be avoided if, instead of `uiomove`, `uiomove_frombuf` is used, because this latter function already performs the check by its own. As stated in `uio(9)`:

> `The uiomove_frombuf() function is a convenience wrapper around uiomove() for drivers that serve data which is wholly contained within an existing buffer in memory. It validates the uio_offset and uio_resid values against the size of the existing buffer, handling short transfers when the request partially overlaps the buffer. When uio_offset is greater than or equal to the buffer size, the result is success with no bytes transferred, effectively signaling EOF.`

This also confirms what is the condition corresponding to EOF. Here the "buffer" consists only of a variable contents, `random_out`, but probably the same applies. With

```
static int
rolld_read(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
        uprintf("Hello.\n");
        uprintf("uio_offset: %ld\n", uio->uio_offset);
        uprintf("uio_resid: %ld\n", uio->uio_resid);

        uint32_t random_out;
        uint32_t random_item;  

        random_item = arc4random();
        random_out = random_item % d_size;

        return (uiomove_frombuf(&random_out, sizeof(random_out), uio));
}
```

the following output is produced:

```
$ cat -v /dev/rolld 
Opened device "rolld" successfully.
Hello.
uio_offset: 0
uio_resid: 4096
Hello.
uio_offset: 4
uio_resid: 4096
Closing device "rolld".
^D^@^@^@
```

The 3 undesired `^@` are due to the fact that `random_out` is declared as `uint32_t`, so they are padding data for the 3 unused bytes, being `random_item % d_size` just 1 byte wide, but `sizeof(random_out)` has value `4` (corresponding to `uint32_t`). The first `read(2)` causes `uiomove_frombuf` to read `4` bytes. Only 1 byte inside `random_out` is not empty; the other ones are `NULL`, that is `^@`. The second `read(2)` call starts in fact not from `uio_offset` `1` as before, but `4`.

Using just `1` as second argument for `uiomove_frombuf` will generate an output with exactly the same structure as before with `uiomove`:

```
return (uiomove_frombuf(&random_out, 1, uio));
```

```
$ cat -v /dev/rolld 
Opened device "rolld" successfully.
Hello.
uio_offset: 0
uio_resid: 4096
Hello.
uio_offset: 1
uio_resid: 4096
Closing device "rolld".
^C
```

### Third version

Using the most relevant results just obtained and the suggestions of the [mentioned post][post-1], a probably better version of the roll device can be created as follows.

The use of a buffer as in the First version may be overkill, but it is also uncomfortable to use the raw value of `random_out`. A simple array of `char`, just inside `rolld_read` may represent the best compromise. Is this array recreated each time `rolld_read` is called? (This would be unefficient with respect to a single buffer allocated with `malloc` just once and in a fixed position, as in the First version).

The code is available [here][rolld-repo-3]. It is minimal as in the second example. The `rolld_read` is:

```
static int
rolld_read(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
	char random_out[2];
	uint32_t random_item = arc4random();

	random_out[0] = 49 + (char)(random_item % d_size);
	random_out[1] = '\n';

	return (uiomove_frombuf(random_out, sizeof(random_out), uio));
}
```

The `arc4random()` value is translated to an ASCII digit between `1` and `6`, so becoming a human-friendly output.

Note that

```
random_out[0] = 49 + (char)(random_item % d_size);
```

is equivalent to

```
random_out[0] = '0' + (char)(random_item % d_size);
```

being the `char` value `0` equal to the `int` value `49`.

As regards the casting precedence over addiction, see [this post][stack-2].

`uiomove_frombuf` with the size of the `char` array is used. The `error` variable is not necessary: it is enough to just return the value of `uiomove_frombuf`, which is `0` upon success (see `uio(9)`).

The output is:

```
$ cat rolld 
3
$ cat rolld 
2
$ cat rolld 
2
$ cat rolld 
2
$ cat rolld 
4
```



[arch-handbook]: https://docs.freebsd.org/en/books/arch-handbook/
[char-device]: https://docs.freebsd.org/en/books/arch-handbook/driverbasics/#driverbasics-char
[makefile-structure]: https://docs.freebsd.org/en/books/arch-handbook/driverbasics/#_makefile
[rolld-repo]: https://github.com/rockyhotas/freebsd_tests/tree/main/kernel_modules/rolld_1
[rolld-repo-2]: https://github.com/rockyhotas/freebsd_tests/blob/main/kernel_modules/rolld_2/rolld.c
[rolld-repo-3]: https://github.com/rockyhotas/freebsd_tests/blob/main/kernel_modules/rolld_3/rolld.c
[stack-1]: https://unix.stackexchange.com/a/323759
[thread-1]: https://lists.freebsd.org/archives/freebsd-hackers/2024-April/003142.html
[post-1]: https://lists.freebsd.org/archives/freebsd-hackers/2024-April/003146.html
[post-2]: https://lists.freebsd.org/archives/freebsd-hackers/2024-April/003145.html
[stack-2]: https://stackoverflow.com/a/3354479
