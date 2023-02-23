# Using the Developer Kit Offline

In order to use the seL4 developer kit offline, for example in an environment where internet access is restricted or unavailable (e.g. within a security domain), a number of dependencies must be downloaded for offline use. This appendix assumes that an internet-connected machine is available, which can be used to download resources that are then used offline by the host machine and build environment.

Note that the offline machine still needs the applications described in the [Software Requirements](../software_requirements.md) section (e.g. Docker and CoolTerm). Installing generally-available applications onto an offline machine is not specific to the developer kit, and it is assumed that the user is able to find a suitable method for their environment.

This appendix covers:

- [Downloading the Docker image to a file](#downloading-the-docker-image-to-a-file)
- [Downloading source code for offline use](#downloading-source-code-for-offline-use)
- [Building U-Boot offline](#building-u-boot-offline)
- [Downloading the pre-prepared SD card images](#downloading-the-pre-prepared-sd-card-images)

## Downloading the Docker image to a file

The Docker image contains all the required toolchain for building seL4 and seL4 applications. Usually the command `docker pull` is used to download an image into the Docker desktop application; however, this will not be possible on a computer without internet access. The following steps detail how the Docker image can be downloaded and saved to a file that can then be transferred to an offline machine.

1. On a machine with internet access and Docker installed, run:
  
    ```bash
    docker pull ghcr.io/sel4devkit/maaxboard:latest
    ```

    The image will now be downloaded. During download of the image, progress is reported in the following format:

    ```bash
    % docker pull ghcr.io/sel4devkit/maaxboard:latest
    latest: Pulling from sel4devkit/maaxboard
    0e29546d541c: Pull complete 
    c2db25fafa50: Pull complete 
    00af15d3de8e: Downloading [==========>                                ]  241.7MB/1.188GB
    c6749363a298: Download complete 
    521655c15371: Download complete 
    ```

2. Once the download has completed, the image can be saved to a file, using `docker save`. Because the image file produced will be quite large (around 4GB), it is recommended that the file is compressed before transferring it to the machine where it will be used offline. This can be done directly from the `docker save` command by piping the output to a compression utility such as `gzip`[^1] as follows:

    ```bash
    docker save ghcr.io/sel4devkit/maaxboard:latest | gzip > sel4devkit_docker.tar.gz
    ```

    Otherwise, without compressing, the command is:

    ```bash
    docker save ghcr.io/sel4devkit/maaxboard:latest > sel4devkit_docker.tar
    ```

    The saved image can now be transferred to the offline host machine.

[^1]: `gzip` is available on macOS and Linux, but not by default on Windows; it is assumed that a Windows user can find a suitable option.

3. On the offline host machine, decompress the image (if applicable) and save the tar file in an appropriate location. From the terminal, navigate to the folder where the tar file is saved, and run:

    ```bash
    docker load -i sel4devkit_docker.tar
    ```

    (As per the introductory section of this appendix, this requires that Docker is installed on the offline host machine.)
    
    Docker will now load the image from the tarball. This may take some time and it may appear that Docker has frozen, but once Docker has loaded the file, the following output should be seen:

    ```bash
    Loaded image: ghcr.io/sel4devkit/maaxboard:latest
    ```

4. The docker image can now be used as detailed throughout the documentation; for more information, see [Build Environment Setup](../build_environment_setup.md).

## Downloading source code for offline use

The source code for seL4 and the seL4 developer kit is stored across multiple repositories. The `repo` command line tool is used alongside manifest files to make downloading and managing the code from these multiple repositories easier. A manifest file details what repositories and versions of those repositories are required for a certain project. There are two main manifests for the seL4 developer kit:

- `camkes-manifest` - used to download the source for building CAmkES applications;
- `sel4test-manifest` - used for downloading the source for building the seL4Test program for the Avnet MaaXBoard.

In order to download source code for offline use, an internet-connected computer is required with the `repo` tool installed. (See [here](https://gerrit.googlesource.com/git-repo/) for details on how to install `repo`.)

1. From the internet-connected computer, create a new folder to store the source code and navigate into it from the command line.

2. In the new folder, to download the source for `camkes-manifest`, run the following command to initialise the `repo` tool in this folder:

    ```bash
    repo init -u https://github.com/sel4devkit/camkes-manifest.git
    ```

    A similar command would be used to download the source for `sel4test-manifest`:
    
    ```bash
    repo init -u https://github.com/sel4/sel4test-manifest.git
    ```

Note that Repo might request a Github username and email. For read-only access, any string can be provided, but for write access a Github account will be required.

3. Once the initialisation has completed, a synchronisation must be performed in order to download the source code from the various repositories. To do this run `repo sync` from the same folder.

4. Once the `repo sync` has completed, all the required source files will be contained within the folder, and this folder can  be transferred to the offline machine. Instructions for building can be followed exactly as detailed in the rest of the documentation.

5. To download updated source code, the folder can be transferred back to a machine with internet access and the `repo` tool, and `repo sync` can be run again.

## Building U-Boot offline

1. On a computer with an internet connection, clone the maaxboard-uboot repository using:

    ```text
    git clone https://github.com/sel4devkit/maaxboard-uboot.git
    ```

2. Run the `clone.sh` script to download the required files for building U-Boot.

3. When the clone is complete, the following message is shown:

    ```text
    +----------------------------------------------------------------+
    |  Source code for compiling U-Boot now cloned.                  |
    |  This folder can now be transferred to an offline computer     |
    |  and U-Boot built by running build-offline.sh.                 |
    |  If you ran build.sh, build will start now.                    |
    +----------------------------------------------------------------+
    ```

4. The cloned repository folder can be transferred to an offline machine, and built within the Docker container using `build-offline.sh`.

## Downloading the pre-prepared SD card images

Two SD card images are available for the MaaXBoard as part of the developer kit: one contains a precompiled U-Boot image with a `BOOT` partition for holding seL4 executables and a `FILESYS` partition for use as a filesystem; and the other contains the above in addition to a precompiled `seL4Test` executable. These can be downloaded from the `disk_images` folder of the [maaxboard-prebuilt](https://github.com/sel4devkit/maaxboard-prebuilt/tree/master/disk_images) repository. Instructions for writing these to an SD card can be found in the [SD Card preparation](../sd_card_preparation.md) section.
