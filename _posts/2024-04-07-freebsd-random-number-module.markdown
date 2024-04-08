---
layout: post
title:  "Example FreeBSD module with a random number generator output"
date:   2024-04-08 20:24:00 +0200
categories: freebsd, kernel, modules
---
**Disclaimer**: this is not an authoritative guide; it is just my understanding of this topic.

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

Without this check, `cat /dev/rolld` continuously prints the buffer contents and for some reason `tail -c 1 /dev/rolld` produces no output, as well as `tail -c 1 /dev/random`.

```
        if ((error = uiomove(rolldmsg->msg, amt, uio)) != 0)
                uprintf("uiomove failed!\n");

        return (error);
}
```

This actually prints the digit resulting from the die roll to the user. Finally,

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


[arch-handbook]: https://docs.freebsd.org/en/books/arch-handbook/
[char-device]: https://docs.freebsd.org/en/books/arch-handbook/driverbasics/#driverbasics-char
[makefile-structure]: https://docs.freebsd.org/en/books/arch-handbook/driverbasics/#_makefile
[rolld-repo]: https://github.com/rockyhotas/freebsd_tests/tree/main/kernel_modules/rolld_1
