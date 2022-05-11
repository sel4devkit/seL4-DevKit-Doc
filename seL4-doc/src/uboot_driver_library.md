# U-Boot Driver Library Overview

To achieve the goals specified in [device driver introduction](device_driver_intro.md) section an extensible library of device drivers ported form U-Boot has been created. This library provides an extensive set of drivers for the Avnet MaaXBoard platform as a worked example of its usage supported by documentation in the following sections on usage and maintenance of the library and guidance on the extension of the library to support additional platforms and devices.

The remainder of this section provides information on the design and structure of the library. Before attempting to

## Design Summary

The primary goal of the library is to allow drivers from U-boot to be used within seL4 with minimal or no code changes. To support this goal the library is comprised of:

- A set of U-Boot drivers.

- The [Driver Model](https://u-boot.readthedocs.io/en/latest/develop/driver-model/index.html) driver framework and its associated subsystems required to support device drivers.

- Stubbed versions of U-Boot subsystems providing a compatibility / conversion layer between the U-Boot source code and underlying seL4 libraries.

- A wrapper around the U-Boot code to provide an API for users of the library to interact with devices and manage library initialisation / shutdown.

## Detailed Design Information

The following sections provide details on the design of stub or wrapper elements that are significant to support the functioning of the drivers within the Driver Model framework.

### Linker Lists

To allow variable levels of functionality to be built into a U-Boot executable, U-Boot utilises a concept of [linker-generated arrays](https://u-boot.readthedocs.io/en/latest/api/linker_lists.html). These are arrays of objects (e.g. drivers, supported commands, driver classes, etc) which are stored by U-Boot within dedicated linker sections to be queried at runtime.

Access to the objects stored within these linker sections is provided by U-Boot through a set of macros defined in the ```linker_lists.h``` header file.

To mimic this functionality within the library the following approach has been taken:

1. Rather than storing the objects within linker sections of the executable, a global variable named ```driver_data``` is declared within ```driver_data.h``` which contains the equivalent object arrays.

2. During initialisation of the library, the platform-dependant contents of the ```driver_data``` global variable are set up.

3. A stubbed version of the ```linker_lists.h``` header file is provided. Rather than accessing data from the executable's linker sections as performed by the original version, the stubbed version accesses data from the arrays within the ```driver_data``` global variable.

### Global Data

U-Boot maintains a [global data](https://u-boot.readthedocs.io/en/latest/develop/global_data.html) structure for the storage of globally required fields.

The macro ```DECLARE_GLOBAL_DATA_PTR``` is used by U-Boot to provide a pointer to this data structure. On most architectures this pointer is stored in a specific register (e.g. register ```r25``` on ARM), such a mechanism is not compatible with seL4.

Within the library the following approach has been taken to global data:

1. A stub of the ```global_data.h``` header file has been provided to provide macro ```DECLARE_GLOBAL_DATA_PTR``` as a standard pointer to a global variable.

2. At initialisation of the library, memory for the global data structure is allocated and initialised as follows:
   - Most elements are unused and set to null values.
   - The global status ```flags``` are set up to mimic an instance of U-Boot that has been relocated and is executing from RAM.
   - Pointers to the flattened device tree are setup to point to seL4's device tree.
   - A [live device tree](https://u-boot.readthedocs.io/en/latest/develop/driver-model/livetree.html) is built from the flattened device tree and referenced.

### Timer

The U-Boot code expects to have access to have access to a high-frequency, monotonic, timer. The timer is accessed by the U-Boot source code through calls to routines ```get_ticks```, ```get_timer_*``` and ```timer_get_*```.

Provision of such a timer is platform dependent. For the Avnet MaaXBaord the timer has been implemented through use of the System Counter (SYS_CON) device provided by the iMX8MQ SoC, see file ```timer_imx8mq.c``` for details.

### Memory Mapped IO

One of the core mechanisms for providing a software interface to a hardware device is through the use of [memory mapped IO](https://en.wikipedia.org/wiki/Memory-mapped_I/O).

The memory addresses which a U-Boot driver uses for communication with a device can be either read from the device tree (i.e. from a devices ```reg``` property) or in some cases can be hard-coded into the driver. At this pointed it should be noted that U-Boot source code is intended to be executed in an environment with direct access to the system's physical address space whilst, however the library is executed within a virtual address space provided by seL4.

As such the addresses used by U-Boot drivers are *physical* addresses which are not directly accessible by the library. To work around this issue:

1. During library initialisation the list of all platform-dependent devices that need to be accessed must be provided. For each device:
   1. The device's physical address range is read from the device tree.
   2. The device's physical address range is mapped into the virtual address space through use of the IO mapping routines provided by seL4's platform support library.
   3. The mapping between the physical address range and the mapped virtual address range is stored within the ```sel4_io_map``` package (part of the libraries wrapper functionality).

2. When a driver attempts to perform memory mapped IO it calls architecture-dependent routines from the ```io.h``` header, e.g. ```readl``` or ```writel```. A stubbed version of the ```io.h``` header is provided that uses the ```sel4_io_map``` package to translate the physical addresses provided by the driver to the equivalent address mapped into the libraries virtual address space.

U-Boot drivers performing memory mapped IO should perform seamlessly without the need for any modifications to the driver.

### DMA

TBC

- Drivers requiring DMA may need to be modified. For XHCI USB all of the required modification have been made in the XHCI stack so other drivers should not require changes. Drivers using the DMA interface exported by linux/dma-mapping.h should work; an seL4 specific implementation of this API has been provided by the library. Drivers not using either of these mechanisms will need to be updated to use the API exported by sel_dma.h (see the fec_mxc.c ethernet driver as an example).

### Console

A minimal stub of U-Boot's console subsystem has been provided (see ```console.c```). This stub provides the subset of functionality required to allow input and output devices to be registered and accessed by the driver library.

It should be noted that the console systems's ```stdout``` file is not used by the library; instead all output is routed directly to C libraries output routines.

The console subsystem's ```stdin``` file, however, is used. For example, if a USB keyboard is registered with the console as the ```stdin``` device then subsequent calls to retrieve input from ```stdin``` will return keypresses input from the USB keyboard.

### Standard Output and Logging

The wrapper header file ```uboot_print.h``` provides a set of macros that map:

- All U-Boot standard output routines onto calls to the C libraries ```printf``` routine.
- All U-Boot logging routines onto the seL4 platform support libraries's ```ZF_LOG*``` routines at an equivalent logging level.

### Initialisation

As part of library initialisation any U-Boot subsystems that require explicit initialisation are handled (see ```uboot_wrapper.c:initialise_uboot_wrapper```). This mimics the minimal subset of the initialisation that would normally be performed by [U-Boot's start-up routine](https://github.com/u-boot/u-boot/blob/master/common/board_r.c).

Initialisation of the library comprises:

- Initialisation of the memory mapped IO and DMA wrappers.
- Initialisation of the monotonic timer.
- Initialisation of the Linker Lists data structure.
- Initialisation of the Global Data data structure.
- Initialisation of U-Boot's Environment subsystem (manages storage of environment variables).
- Initialisation of U-Boot's Driver Model subsystem.
- Initialisation of U-Boot's MMC subsystem (if SD/MMC drivers are used by the platform).
- Initialisation of U-Boot's Network subsystem (if Ethernet drivers are used by the platform).

### Configuration

TBC

## Code Structure

TBC

- General structure:
  - Code structure
  - U-Boot code, minimally modified.
  - Stub
    - delay / time.
    - print / debug.
  - Wrapper
    - Mimic of u-boot start-up code.
    - Memory mapping.
    - DMA.

## Limitations of the Library

TBC

- Not thread safe ( uses musclib that are not thread safe).
- Don’t expect amazing performance.
- Unable to control ARM power domains (requires calls to the ATF).
- Only works with drivers that have been updated to work with U-Boot’s “live tree” functionality. Such for uses of devfdt_xxx routines and replace with the equivalent dev_xxx routine. See pinctrl-imx.c for an example.
- Interrupts are currently not used / supported.