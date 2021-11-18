---
layout: post
title:  "Different approaches in character device creation with a Linux kernel module"
date:   2021-11-16 22:56:12 +0100
categories: linux, kernel, modules, device
---
**Disclaimer**: this is not an authoritative guide; it is just my understanding of this topic.

In the repository [rpi_simple_driver_examples][rpi-rep], sourcecode files [02/dev_nr.c][rpi-file1] and [03/read_write.c][rpi-file2] use two different strategies to create a Linux character device file.

- [02/dev_nr.c][rpi-file1] uses the kernel API `register_chrdev()` (defined in [`include/linux/fs.h`][fs-h]), which is considered the *old way* to register a character device;
- [03/read_write.c][rpi-file2] uses the kernel API `alloc_chrdev_region()` (defined in [`fs/char_dev.c`][char_dev-c]).

All the paths are referred to the kernel sourcecode root directory. The kernel uses structures of type `struct cdev` to represent character devices internally. This `struct` is defined in [`include/linux/cdev.h`][cdev-h] as:

```
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;
	dev_t dev;
	unsigned int count;
} __randomize_layout;
```

In order to use a character device, the driver must first allocate and register one or more of these structures. Therefore, driver code should include `<linux/cdev.h>`, where not only the structure, but also its associated helper functions are declared. `__randomize_layout` is a macro, which is out of the scope of these considerations. Two relevant members are:

- `const struct file_operations *ops`, which relates the ordinary operations (`open`, `read`, `write`, ...) on the character device file represented by this structure to some internal operations in the kernel module;
- `dev_t dev`, which stores the (major, minor) numbers couple representing the character device file corresponding to this structure.

The helper functions declared in [`include/linux/cdev.h`][cdev-h] are defined in [`fs/char_dev.c`][char_dev-c], with some useful code comments:

- `void cdev_init(struct cdev *cdev, const struct file_operations *fops)`. This functions receives two parameters: a pointer to a `struct cdev` and a pointer to a `struct file_operations`. The `struct cdev` is initialized and, in particular, its member `ops` is set to the value of `fops`, so that the `struct cdev` becomes related to those file operations. This `struct cdev` is then ready for `cdev_add()`.
- `struct cdev *cdev_alloc(void)`. Allocate a `cdev` structure in memory, partially initialize it and return a pointer to it.
- `int cdev_add(struct cdev *p, dev_t dev, unsigned count)`. Add the device represented by `p` to the system, creating a character file in `/dev`. `dev` is the first (lowest) couple of (major, minor) numbers related to the device and `count` is the number of consecutive minor numbers corresponding to the device.
- `void cdev_del(struct cdev *p)`. Remove the device previously created by `cdev_add`, *possibly freeing the structure `p`*. This, however, only guarantees that the character device *will no longer be able to be opened*, but any character devices already open will remain and their `file_operations` *will still be callable* even after `cdev_del` returns.

`cdev_alloc` is useful when a simple `struct cdev` pointer is declared in the code. Then, when `cdev_alloc` is invoked, it immediately allocates space in memory for the `struct cdev`:

    struct cdev *my_cdev_pointer;
    my_cdev_pointer = cdev_alloc();

`cdev_init` assumes instead that a brand new `struct cdev` is already present in memory and must simply be initialized. It is used when the whole `struct cdev` is declared:

    struct cdev my_cdev;
    ...
    cdev_init(my_cdev, my_fops);
    my_cdev.owner = THIS_MODULE;

Note that the `owner` member of the structure must be set in a separate line, in this case. Of course, the code must previously also set up a `struct file_operations`, whose pointer is `my_fops`, before invoking `cdev_init()`. The difference between `cdev_alloc()` and `cdev_init()`, which seem to perform very similar tasks, is shown later.

The kernel APIs ending with `_region` (`alloc_chrdev_region()` and `register_chrdev_region()`, which is not used in [rpi_simple_driver_examples][rpi-rep]) do not invoke `cdev_alloc()` at all: the driver (represented by the kernel module) has to invoke `cdev_alloc()` or `cdev_init()`. Instead, the kernel API not ending with `_region`, namely `register_chrdev()`, [invoke `cdev_alloc`][register_chrdev-1] and therefore the driver does not need to do so (source: comments to [this question][source-1]).

In particular,

    int register_chrdev(unsigned int major, const char *name, const struct file_operations *fops);

registers minor numbers `0-255` for the major specified in `major`, and **sets up a default `struct cdev` for each**. Drivers using this API **must be prepared to handle open calls on all 256 minor numbers** (whether they correspond to real devices or not), and they cannot use major or minor numbers greater than 255 ([source][source-2]). Moreover, the device file(s) must then be created with `sudo mknod /dev/<name> c <major_number> <minor_number>`. This whole procedure is the old way to create character devices and it is discouraged.

The new method to register a character device into the kernel follows instead two separate steps:

- the allocation of a range of device numbers;
- the association of devices with those numbers.

The first step is accomplished by either:

- `int register_chrdev_region(dev_t from, unsigned count, const char *name)`;
- `int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)`.

They are defined in [`fs/char_dev.c`][char_dev-c]. In the code comments, for both of them it is stated:

> register a range of char device numbers

`register_chrdev_region()`, starting from the (major, minor) couple specified by the `dev_t from`, allocates `count` minor numbers and relate them to `name`. `alloc_chrdev_region()` performs the same operation, but: given the first minor `baseminor`, it chooses autonomously the major number for the starting (major, `baseminor`) couple; then, it allocates `count` minor numbers. The chosen (major, `baseminor`) couple is returned through the pointer `dev`. In both cases, the returned value is `0` on success, or a negative value in case of failure.

The starting (major, minor) couple is totally (in the case of `register_chrdev_region()`) or partially (in the case of `alloc_chrdev_region()`) custom: for this reason, two or more different devices may share the same major number (just as an example: if a region starts at `(15,102)` and ends at `(15,108)`, another driver may theoretically be assigned the region between `(15,109)` and `(15,120)`: two different devices would have the same major number `15`). The minor numbers can be greater than `255`, unlike for `register_chrdev()`, but there is a maximum: if it is overcome, the range extends to the next major number. Device "boundaries" are represented only by the specified ranges: at least, theoretically (in practice, ranges which include more than one device number are not chosen by programmers).

Regardless of the choice between `register_chrdev_region()` and `alloc_chrdev_region()`, device numbers can be released (and made available to the system for new allocations) with `void unregister_chrdev_region(dev_t from, unsigned count)` (also defined in [`fs/char_dev.c`][char_dev-c]).

In order for the device(s) (corresponding to the range just allocated) to actually appear in the filesystem, it is necessary to invoke `device_create()`, defined in [drivers/base/core.c][core-c]:

    struct device *device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...)

The syntax `...` indicates that this is a [variadic function][variadic-function], which accepts a variable number of arguments. It returns a pointer to `struct device` on success, or `ERR_PTR()` (that is a `NULL` pointer) on error. As stated in the comments before this function definition:

> Note: the `struct class` passed to this function must have previously been created with a call to `class_create()`.

Before `device_create()`, it is therefore necessary to create a device class. The function is actually a macro, in [`linux/device/class.h`][class-h]:

```
#define class_create(owner, name)		\
({						\
	static struct lock_class_key __key;	\
	__class_create(owner, name, &__key);	\
})
```

As stated in the comments to the code, `owner` is a *pointer to the module that is to "own" this struct class*, while `name` is a *pointer to a string for the name of this class*. The function can simply be invoked as:

```
#define DRIVER_CLASS "CustomDeviceClass"
static struct class *my_class;
...
my_class = class_create(THIS_MODULE, DRIVER_CLASS);
...
```

Note that `DRIVER_CLASS` and the pointer `my_class` should be accessible by at least the `ModuleInit` and the `ModuleExit` functions in the kernel module, so they must be declared outside them.

The association between a device and the range of its corresponding minor numbers (which has been allocated with `register_chrdev_region()` or `alloc_chrdev_region()`) is realized through the `struct cdev`. This struct represents the device. It can be allocated in two different ways: one using `cdev_alloc()`, one using `cdev_init()`.

A first example is:

```
struct cdev *my_dev;	// this should be put outside any other function
...
static int __init ModuleInit(void) {
	...
	my_dev = cdev_alloc();
	if (my_dev != NULL) {
		my_dev->ops = &my_fops;
		my_dev->owner = THIS_MODULE;
	}
	else
	...
```

Note that the pointer `my_dev` is (and must be) accessible by any function in the kernel module sourcefile: `ModuleInit`, which allocates the `cdev` structure, but also `ModuleExit`, which deletes it. `my_fops` is an already initialized `file_operations` structure. `cdev_alloc()` dynamically allocates the space in memory (in the [heap](source3)) needed by a `struct cdev` and returns the address of the beginning of this space into `my_dev`. This creates a standalone `cdev struct`: a struct which exists by its own.

The second way is the one chosen in [03/read_write.c][rpi-file2]:

```
static struct cdev my_cdev;	// this should be put outside any other function
...
static int __init ModuleInit(void) {
        ...
        cdev_init(&my_cdev, my_fops);
        my_cdev.owner = THIS_MODULE;
        ...
```

When the kernel module represents a device, it may create a specific structure for it, which would include, between its members, *also* a `struct cdev`. When the larger `struct` is created, its memory area already includes the space required for the `struct cdev`: there is no need to allocate memory with `cdev_alloc()`. `cdev_init()` is then used, which relates the `struct cdev` to an already initialized `struct file_operations`; it does not modify the member `owner` of the `cdev struct`, which must be set in a separate line. If the `struct cdev` is just a member of a larger, device-specific `struct`, it is not standalone. However, this method can be used even without the larger `struct`. A `struct cdev` may be declared as a `static` variable (therefore accessible by all the functions in the kernel module sourcecode file). For this reason, it is stored in the [data][source3] section of the kernel module memory area (not the stack, not the heap).

This second way is the most common one to create and initialize a `struct cdev`.

It may seem, from the information provided so far, that `cdev_init()` is also usable in the first example, after `cdev_alloc()`. It is not. `cdev_alloc()` makes several operations which has not been shown here: it partially initializes the `struct cdev`, even if it does not set the `ops` and/or `owner` member. The invocation of `cdev_init()` on a `struct cdev` created with `cdev_alloc()` would (to some extent) break the initialization already provided by `cdev_alloc()`: for this reason, a kernel error message would be generated and the operation is not permitted. For example, `cdev_alloc()` somewhat prepares the kernel to free the memory space of the `struct cdev` when it is no more used: `cdev_init()` would delete this information and the memory would be never released.

If the lifetime of the character device must be the same as the kernel module, the preferable procedure is the second one, with a `static struct cdev my_cdev` (or a global one) and `cdev_init()`.

The `struct cdev` has been created and initialized, but it is not bound to any (major, minor) number range. The final step establishes this relation, using the kernel function

    int cdev_add(struct cdev *p, dev_t dev, unsigned count)

The range extending for `count` contiguous minor numbers and starting from the couple (major, minor) specified by `dev`, which has been previously allocated through either `register_chrdev_region()` or `alloc_chrdev_region()`, is now associated to the `struct cdev` pointed by `p`, which represents the device. This function *can* fail: in this case, it returns a negative value, which the kernel module should be able to handle (at least, printing an error message). If instead the function returns `0`, the device is usable through the character file(s).

Summarizing: the required amount of (major, minor) numbers is reserved; the `struct cdev` is created and initialized; the `struct cdev` is finally connected to the (major, minor) numbers. Now, the device represented by the `struct cdev` can be accessed through one or more files in `/dev`: the ordinary operations on these files (read, write, ...) are translated (through the `struct file_operations`) into specific operations by the kernel module, which acts on the actual device.

To remove the `struct cdev` use:

    void cdev_del(struct cdev *p)

defined in [`fs/char_dev.c`][char_dev-c]. To remove the filesystem devices:

    void device_destroy(struct class *class, dev_t devt)

defined in [`drivers/base/core.c`][core-c]. Note that `devt` is the same parameter `dev` which had been passed to `cdev_add()`.

To remove the class:

    void class_destroy(struct class *cls)

defined in [`drivers/base/class.c`][class-c].

Functions should be called in this order; after them, `unregister_chrdev_region()`.

Another important source for this whole post can be found [here][source-4].



[rpi-rep]: https://github.com/rockyhotas/rpi_simple_driver_examples
[rpi-file1]: https://github.com/rockyhotas/rpi_simple_driver_examples/blob/main/02/dev_nr.c
[rpi-file2]: https://github.com/rockyhotas/rpi_simple_driver_examples/blob/main/03/read_write.c
[fs-h]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/fs.h
[cdev-def]: https://elixir.bootlin.com/linux/v5.10.63/source/include/linux/cdev.h#L14
[cdev-h]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/cdev.h
[char_dev-c]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/char_dev.c
[core-c]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/core.c
[class-h]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/device/class.h
[class-c]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/class.c
[source-1]: https://stackoverflow.com/q/55287522
[source-2]: https://www.makelinux.net/ldd3/chp-3-sect-4.shtml
[source-3]: https://stackoverflow.com/a/14588866
[source-4]: https://lwn.net/Articles/195805/
[register_chrdev-1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/char_dev.c#n280
[variadic-function]: https://en.wikipedia.org/wiki/Variadic_function#In_C
