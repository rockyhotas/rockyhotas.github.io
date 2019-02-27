---
layout: post
title:  "FreeBSD kernel modules"
date:   2019-02-27 11:25:10 +0100
categories: unix
---
FreeBSD kernel comes with some basic features. It could be necessary, however, to widen them with additional drivers or functionalities: for this purpose, kernel modules allow the system administrator to integrate new code inside a running system. In FreeBSD, a module is provided as an [object file] with extension `.ko`. It

> contains code to extend the running kernel, or so-called base kernel [...]. \[Modules\] are typically used to add support for new hardware (as device drivers) and/or filesystems, or for adding system calls. When \[their functionalities are\] no longer required, \[they\] can be unloaded in order to free memory and other resources.

(From [Wikipedia])

To add, list and remove FreeBSD kernel modules, the `kld` interface is available: these operations are respectively performed by the tools `kldload(8)`, `kldstat(8)` and `kldunload(8)`.

<BR>

### kldstat

More specifically, `kldstat` lists the modules currently loaded and in use by the kernel. For example:

    $ kldstat
    Id Refs Address                Size Name
     1   47 0xffffffff80200000  243cd00 kernel
     2    1 0xffffffff8263d000    265d0 ext2fs.ko
     3    1 0xffffffff82665000     2be8 coretemp.ko
     4    1 0xffffffff82821000   154cc8 radeonkms.ko
     5    1 0xffffffff82976000    858b0 drm.ko
     6    4 0xffffffff829fc000    104f0 linuxkpi.ko
     7    3 0xffffffff82a0d000    13f30 linuxkpi_gplv2.ko
     8    2 0xffffffff82a21000      6c0 debugfs.ko
     9    1 0xffffffff82a22000      a75 radeon_RV635_pfp_bin.ko
    10    1 0xffffffff82a23000     5573 radeon_RV635_me_bin.ko
    11    1 0xffffffff82a29000      d73 radeon_R600_rlc_bin.ko
    12    1 0xffffffff82a2a000    12343 radeon_R600_uvd_bin.ko
    13    1 0xffffffff82a3d000     1800 uhid.ko
    14    1 0xffffffff82a3f000     23a8 ums.ko
    15    1 0xffffffff82a42000      acf mac_ntpd.ko
    16    1 0xffffffff82a43000     9c08 fuse.ko

Each module has an `id`. Kernel, and so its modules, are loaded into RAM: therefore, the memory address of the beginning of each module is shown in this output. `Size` is by default an hexadecimal number of bytes: option `-h` can be used to display a human-readable value instead. 

Consider for example module `ext2fs.ko`: it begins at byte `0xffffffff8263d000` in memory, its size is `0x265d0` bytes. Note that the sum

    0xffffffff8263d000 + 0x265d0 = 0xffffffff826635D0

(which denotes the end of the module) is **lower** than the beginning byte `0xffffffff82665000` of the next module `coretemp.ko`. This occurs for each module. In `ext2fs.ko`, the difference amounts to `0x1a30` (`6704` in decimal units) bytes: this value is not constant across modules.

Use options `-vd` to display more information and to list the modules already included in `kernel`.

<BR>

### kldload

`kldload(8)` manpage in FreeBSD provides very exhaustive information about modules and their loading procedure. Their name can be used, with or without the extension `.ko`. It can be provided their full path or just their name: in this latter case, the module will be searched in the paths provided by the kernel parameter `kern.module_path`. To obtain its value, use `sysctl`. For example:

    $ sysctl kern.module_path
    kern.module_path: /boot/kernel;/boot/modules;/boot/dtb;/boot/dtb/overlays

Usually, modules which are part of the base system (shipped with the OS) are in `/boot/kernel`, while modules installed by ports or packages, or proprietary ones, are in `/boot/modules`.

For example, `ext2fs.ko`, `coretemp.ko`, `fuse.ko` are stored in `/boot/kernel`, while `radeonkms.ko`, `drm.ko` and the modules `radeon_R*.ko` are stored in `/boot/modules`.

`kldload(8)` also states that

    To load a module from the current directory it must be specified as a full
    or relative path.	

To load the example module `ext2fs.ko`, which is already in `/boot/kernel`, use

    # kldload ext2fs.ko

<BR>

### kldunload

As anticipated, each kernel module requires an amount of RAM, according to its size. If its functionality is no more required, a module can be removed from the kernel, so freeing its related portion of memory. For example:

    # kldunload ext2fs.ko

See `kldunload(8)` for more options.

<BR>

### Load at boot

Modules can be loaded at any moment with `kldload(8)`, but if their use is always required, they can alternatively be loaded at boot time. Two methods are available, with a different syntax.

* In `/etc/rc.conf`, add the following line:

    `kld_list="/full/path/to/module.ko"`

* In `/boot/loader.conf`, add the following line:

    `module_name_without_ko="YES"`

As specified in [this message], both of them are «correct»; the first method, however, may be preferred because, according to `rc.conf(5)`, `kld_list` specifies «a list of kernel modules to load *right after the local disks are mounted*». Not only: «loading modules at this point in the boot
process is **much faster** than doing it via `/boot/loader.conf`», but obviously they can not be modules necessary to mount a local disk.

If for example modules `ext2fs` and/or `fuse` are required to mount one or more local disks (or partitions), they must be loaded using `/boot/loader.conf`.


[object file]: https://en.wikipedia.org/wiki/Object_file
[Wikipedia]: https://en.wikipedia.org/wiki/Loadable_kernel_module
[this message]: https://lists.freebsd.org/pipermail/freebsd-questions/2014-August/260678.html
