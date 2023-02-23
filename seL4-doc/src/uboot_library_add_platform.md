# Library Extension - New Platform

This section documents the required actions and guidance to add support for a new platform to the library.

By the end of this section, an seL4 executable will be built for the new platform that can initialise the library (although the library may not support any of the platform's devices). Later sections of this guide cover the required actions to add driver support into the library for the new platform (e.g. see the [Odroid-C2 worked example appendix](appendices/add_odroidc2.md)).

Throughout the sections of this guide devoted to extension of the U-Boot driver library, it is expected the developer is working within the folder structure created by the `repo` tool (e.g. as used to build the [test applications](uboot_driver_usage.md)). Key folders and files within the hierarchy are shown below:

```text
<manifest root>
|
├───kernel
│   └───tools
│       └───dts
│
└───projects
    ├───camkes
    │   └───apps
    │       └───uboot-driver-example
    │           └───include
    │               └───plat
    │                   └───<platform name>
    │                       └───platform_devices.h
    │
    ├───projects_libs
    │   └───libubootdrivers
    │       ├───include
    │       │   └───plat
    │       │       └───<platform name>
    │       │           └───plat_driver_data.h
    │       ├───src
    │       │   └───plat
    │       │       └───<platform name>
    │       │           └───plat_driver_data.c
    │       └───CMakeLists.txt
    │
    └───uboot
```

- `kernel/tools/dts`: Location of platform device trees.
- `projects/project_libs/camkes/apps/uboot-driver-example`: [The test application](uboot_driver_usage.md).
- `projects/project_libs/libubootdrivers`: Referred to as "the library" throughout. See [linked Git repository](https://github.com/sel4devkit/projects_libs/tree/master/libubootdrivers).
- `projects/uboot`: Fork of the U-Boot project source code (note, this is also symlinked to `projects/project_libs/libubootdrivers/uboot`).

## Required Reading

The developer should be familiar with the following tools and concepts in order to add platform and driver support to the library.

- *Device Tree*: An introduction to device tree data is [provided by the Linux documentation](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html) and by the [Zephyr project](https://docs.zephyrproject.org/2.6.0/guides/dts/intro.html).

- *CMake*: Reference documentation is provided by [the CMake project](https://cmake.org/cmake/help/latest/), including details of the structure and syntax of the CMakeLists.txt file.

- *CAmkES*: The seL4 foundation provides [documentation on its use of CAmkES](https://docs.sel4.systems/projects/camkes/) including tutorials.

## Add basic support to library

To allow the library to be successfully compiled for a new platform, the following changes will need to be made to the library.

### Update the library's CMake file to support the platform

The library's CMake file (located at `projects/project_libs/libubootdrivers/CMakeLists.txt`) contains a section titled `Platform specific settings` to control the settings for each platform. This section:

1. Declares a set of variables to control which drivers and optional capabilities are to be built for each platform. The default values produce a build including only a dummy timer driver; this is the minimum necessary to allow the library to be built.

2. Provides a conditional block of settings associated with each supported platform, e.g. to control which of the supported drivers to build. If no conditional block is supplied for a platform, the compilation of the library fails and an error message is returned.

The minimal change to this section that allows the library to be built is therefore the inclusion of a conditional block for the platform, even if that section makes no changes to the default settings. For example, to add support for a platform that is named  `foo` within the seL4 build system, the following changes would be required:

```makefile
    # Set up the applicable drivers and platform dependent configuration.
    if(KernelPlatImx8mq)
        ...
+   elseif("${KernelPlatform}" STREQUAL "foo")
+       # Platform specific settings for the Foo board.
    else()
        message(FATAL_ERROR "Unsupported platform. Aborting.")
    endif()
```

Note that where settings are shared across multiple platforms, this logic can be simplified to reflect the shared settings. An example of this is provided in the library's `CMakeLists.txt` for the `maaxboard` and `imx8mq-evk` platforms, which both use the iMX8MQ SoC.

It should also be noted that U-Boot drivers can support multiple devices. For example, many of the drivers added to support the Avnet MaaXBoard support multiple iMX SoCs. If previously supported drivers are compatible with the new platform, this support can be added now; for example, if platform `foo` has a GPIO device supported by the `gpio_mxc` driver (as used by the Avnet MaaXBoard), then the line `set(gpio_driver "gpio_mxc")` could be added to the `foo` platform's settings to enable support.

To enable access to platform specific header files it is necessary to create a symlink within the U-Boot code structure named `arch` to one of the many `arch-xxx` folders provided by U-Boot; this mimics an equivalent action taken by U-Boot's native build system. As an example it can be seen that for the iMX8MQ based platforms the `arch` symlink points to the `arch-imx8m` folder. It is expected that identification of the correct folder to link to will in most cases will be possible through naming convention alone. To definitively determine the folder to link, the symlink created during a build of U-Boot could be inspected.

### Define platform specific Linker Lists data structure

As documented within the [Linker Lists section of the library overview](uboot_driver_library.md#linker-lists), the library requires a global data structure named `driver_data` to be declared to allow the U-Boot source code to access optional functionality included in the executable.

To define `driver_data` for the new platform, new platform-specific files must be added to the library. The code blocks that follow provide minimal templates for these files that include the base device classes, drivers for those classes, and base commands that are required by all platforms.

File `include/plat/foo/plat_driver_data.h`:

```c
/*
 * This file defines which drivers, driver classes, driver entries and commands
 * are to be included in the compiled library for this platform.
 *
 * This allows only the drivers compatible with the targeted platform to be
 * included with all non-compatible drivers excluded.
 *
 * It should be noted that some of these are fundamental to allowing the U-Boot
 * driver model to function (e.g. the nop, root and simple bus drivers).
 */

/* Define the number of different driver elements to be used on this platform */
#define _u_boot_uclass_driver_count     5
#define _u_boot_driver_count            2
#define _u_boot_usb_driver_entry_count  0
#define _u_boot_part_driver_count       0
#define _u_boot_cmd_count               3
#define _u_boot_env_driver_count        0
#define _u_boot_env_clbk_count          0
#define _u_boot_driver_info_count       0
#define _u_boot_udevice_count           0

/* Define the uclass drivers to be used on this platform. The count of declarations
 * in this section must match the value of _u_boot_uclass_driver_count */
extern struct uclass_driver _u_boot_uclass_driver__nop;
extern struct uclass_driver _u_boot_uclass_driver__root;
extern struct uclass_driver _u_boot_uclass_driver__simple_bus;
extern struct uclass_driver _u_boot_uclass_driver__phy;
extern struct uclass_driver _u_boot_uclass_driver__blk;

/* Define the drivers to be used on this platform. The count of declarations
 * in this section must match the value of _u_boot_driver_count */
extern struct driver _u_boot_driver__root_driver;
extern struct driver _u_boot_driver__simple_bus;

/* Define the driver entries to be used on this platform. The count of declarations
 * in this section must match the value of _u_boot_usb_driver_entry_count */

/* Define the disk partition types to be used. The count of declarations
 * in this section must match the value of _u_boot_part_driver_count */

/* Define the u-boot commands to be used on this platform. The count of declarations
 * in this section must match the value of _u_boot_cmd_count */
extern struct cmd_tbl _u_boot_cmd__dm;
extern struct cmd_tbl _u_boot_cmd__env;
extern struct cmd_tbl _u_boot_cmd__setenv;

/* Define the u-boot environment variables callbacks to be used on this platform. The
 * count of declarations in this section must match the value of _u_boot_env_clbk_count */
```

File `src/plat/foo/plat_driver_data.c`:

```c
#include <uboot_helper.h>
#include <driver_data.h>

#include <dm/device.h>
#include <dm/uclass.h>
#include <dm/platdata.h>
#include <usb.h>
#include <part.h>

void initialise_driver_data(void) {
    /* The number of elements in the uclass_driver_array must match the
     * _u_boot_uclass_driver_count constant (see plat_driver_data.h) */
    driver_data.uclass_driver_array[0]  = _u_boot_uclass_driver__nop;
    driver_data.uclass_driver_array[1]  = _u_boot_uclass_driver__root;
    driver_data.uclass_driver_array[2]  = _u_boot_uclass_driver__simple_bus;
    driver_data.uclass_driver_array[3]  = _u_boot_uclass_driver__phy;
    driver_data.uclass_driver_array[4]  = _u_boot_uclass_driver__blk;

    /* The number of elements in the driver_array must match the
     * _u_boot_driver_count constant (see plat_driver_data.h) */
    driver_data.driver_array[0]  = _u_boot_driver__root_driver;
    driver_data.driver_array[1]  = _u_boot_driver__simple_bus;

    /* The number of elements in the cmd_array must match the
     * _u_boot_cmd_count constant (see plat_driver_data.h) */
    driver_data.cmd_array[0]  = _u_boot_cmd__dm;
    driver_data.cmd_array[1]  = _u_boot_cmd__env;
    driver_data.cmd_array[2]  = _u_boot_cmd__setenv;
}
```

As support for additional optional objects (e.g. drivers, driver classes, commands, etc.) are added for a platform, these files will need to be updated to reference the associated objects. See the files for the Avnet MaaXBoard platform for an example of these how these files provide more extensive support.

## Add support to example application

Section [Using the U-Boot Driver Library](uboot_driver_usage.md) introduced the demonstration application  `uboot-driver-example`. This section documents the changes required to the `uboot-driver-example` application to allow it to be built for the new platform.

The following empty template file needs to be added to the application `include/plat/foo/platform_devices.h`:

```c
#pragma once

/* List the set of device tree paths that include the 'reg' entries
 * for memory regions that will need to be mapped */
#define REG_PATHS {};
#define REG_PATH_COUNT 0

/* List the set of device tree paths for the devices we wish to access.
 * Note these need to be the root nodes of each device to be accessed */
#define DEV_PATHS {};
#define DEV_PATH_COUNT 0

/* Provide the hardware settings for CAmkES. Note that we only need to inform
 * CAmkES of the devices with memory mapped regions, i.e. the REG_xxx
 * devices. See https://docs.sel4.systems/projects/camkes for syntax */

#define HARDWARE_INTERFACES

#define HARDWARE_COMPOSITION

#define HARDWARE_CONFIGURATION
```

As support for devices is added for a platform, this file will need to be updated to reference those devices from the platform's device tree. See the file for the Avnet MaaXBoard platform for an example that provides more extensive support.

## Guidance on next steps

At this point, the library should compile cleanly for the new platform but will fail during library initialisation due to no devices being supplied.

The order in which drivers need to be added is likely to depend on the intended usage of the library; e.g. support for a single device or support for multiple devices, and the internal needs of the device drivers to be added.

The following drivers are likely to form the core of the support for a platform and underpin the capabilities of other drivers. It may be possible to support some devices without these core drivers, but that would need to checked on a case-by-case basis.

### Timer

As documented [in the library overview](uboot_driver_library.md#timer), some U-Boot drivers rely upon access to a monotonic timer to underpin their timing needs.

By default a platform will be supported by the `dummy` timer driver (see library file `src/timer/timer_dummy.c`). Whilst the dummy timer driver allows the library to build cleanly, it will raise an assertion error / exception, thereby halting execution, should any of the timing routines be used. Some simple drivers (e.g. GPIO drivers) typically do not require any timing routines and so can be supported without the need to provide a functional timer; however, a functional timer will typically need to be provided to support more complex devices such as Ethernet or USB.

The means of providing a timer driver is architecture- and platform-dependent, so detailed guidance cannot be provided. A worked example of a timer driver for the iMX8MQ SoC is provided in file `src/timer/timer_imx8mq.c`; it is expected that this driver could be extended to cover multiple iMX SoCs.

### Clock

The U-Boot `CLK` subsystem is used to enable and reconfigure clocks. To enable the `CLK` subsystem, support for the platform's U-Boot clock driver will need to be added. See the configuration of the clock driver for the Avnet MaaXBoard in the library's `CMakeLists.txt` file as an example.

In many cases it may be possible to support devices without the need to add support for a clock driver. Simple devices, e.g. GPIO, may not use a clock source. Even for more complex devices that use a clock source, it may be possible for the device to function without providing a clock driver; e.g. if the device was previously set up and used by the bootloader prior to entry to seL4 then it is likely that the required clocks have already been configured and enabled.

### Pin Multiplexing (IOMUX)

SoCs typically contain a lot of functionality but a have limited number of pins (or pads). Pin multiplexing is used to configure pins for a specific purpose. An IOMUX driver must be supplied to allow the library to configure the pins from the default configuration.

As with the clock driver, in many cases it may be possible to support devices without the need to add support for an IOMUX driver. For example, if a device was previously set up and used by the bootloader prior to entry to seL4 then it is likely that the required pin configuration has already been configured.

### GPIO

Many drivers rely upon the availability of a GPIO driver to function correctly. For example: an MMC driver may use GPIO for card-detect and write-protect sensing; an Ethernet driver may use GPIO to reset an external PHY; an SPI driver may use GPIO for the chip select signal, etc.

As such it may be necessary to provide a GPIO driver for other drivers to function correctly.

## Appendices

- [Odroid-C2 Worked Example](./appendices/add_odroidc2.md)
