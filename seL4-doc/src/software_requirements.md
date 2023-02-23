# Software Requirements

This developer kit is intended to have minimal software requirements on the host machine by using Docker containment as much as possible. The applications that follow are either needed or assumed on the host machine.

Installation instructions are not reproduced here; please follow the instructions given by the application providers. Configuration of the applications is considered in the [Host Machine Setup](host_machine_setup.md) section.

This guide assumes that the user has sufficiently elevated privileges on the host machine to install and configure the applications; for example on macOS and Linux it is assumed that the user has `sudo` privileges.

## Mandatory Requirements

The following mandatory applications are assumed on the host machine:

- **Container toolset**: Used to provide the build environment. This guide assumes use of [Docker Desktop](https://www.docker.com/products/docker-desktop) for macOS or Windows host environments or [Docker Engine](https://hub.docker.com/search?offering=community&operating_system=linux&q=&type=edition) for Linux-based host environments.

    Docker Engine is licensed under the Apache License Version 2.0, see [here](https://docs.docker.com/engine/) for more information. Docker Desktop is free to use for small companies, education and non-commercial open source projects, see [here](https://www.docker.com/pricing/faq/) for details and pricing.

- **SD card writer application**: Used to write the SD card from which the MaaXBoard boots. [Balena Etcher](https://www.balena.io/etcher/) has been selected due to its availability on macOS, Windows and Linux environments. Alternative means of writing SD cards are available but not supported by this document.

    Balena Etcher is licensed under the Apache License Version 2.0, see [here](https://github.com/balena-io/etcher/blob/master/LICENSE) for more information.

- **Serial terminal application**: Used for serial communication with the target platform. [CoolTerm](https://freeware.the-meiers.org/) has been selected due to its availability on macOS, Windows and Linux environments. Alternative applications are available but not supported by this document.

    CoolTerm is distributed as freeware with no documented limitations on use.

## Optional Requirements

If the user wishes to perform TFTP transfer of executables from the host environment to the target platform (see the [Bootloader](bootloader.md) section) then installation of a TFTP server is required.

Full details of the installation and configuration of a TFTP server for every operating system is considered to be outside the scope of this developer kit, however the following guidance is offered:

- **macOS**: A TFTP server is included in macOS by default. The [TftpServer](https://www.macupdate.com/app/mac/11116/tftpserver) application provides a graphical user interface to simplify use of the in-built server.

- **Linux**: An example setup of a TFTP server is included in the [appendix](appendices/tftpboot_example.md)

- **Windows**: Multiple free TFTP server applications are available, e.g. [SolarWinds TFTP Server](https://www.solarwinds.com/free-tools/free-tftp-server) (distributed as "100% free" with no documented limitations on use) and [Tftpd64](https://pjo2.github.io/tftpd64) (licensed under GNU General Public License, version 2).
