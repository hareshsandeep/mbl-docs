## Tutorial: Updating MBL or applications

### Prerequisites

An MBL device can only be updated if the followings are available:

* The device is running an MBL image that can connect to a Pelion account. [Follow the first section in this series](../getting-started/pelion-accounts-and-certificates.html) to request an account.
* The build artefacts for the image to send as the update payload. See the [build tutorial](../getting-started/tutorial-building-an-image.html) for instructions.
* An internet connection on the device. [Follow the tutorial to set up an internet connection](../getting-started/tutorial-connecting-to-a-network-and-pelion-device-management.html).
* The directory in which the manifest tool was initialized, [as reviewed in the development environment setup](../getting-started/development-environment.html).

    <span class="notes">This *must* be the directory from which the `update_default_resources.c` file was obtained for building MBL.</span>

* A Pelion API key, to use the manifest tool from the command line. Follow the instructions in the [requirements section](..//getting-started/api-keys.html) to obtain an API key (when prompted to select a group to set the API key access level, select **Developers**). Make a note of the API key to use it later; for security reasons, the portal will not display it again.

### Workflow

1. Create an update payload file:

    * **For an application**: Make a `tar` file containing the `.ipk` files for the applications to update.

        Note that the `.ipk` files must be in the `.tar`'s root directory, not a subdirectory.

        For example, to create an update payload file at `/tmp/payload.tar` containing an `.ipk` file with the path `/home/user01/my_app.ipk`, run:
        ```
        user01@dev-machine:~$ tar -cf /tmp/payload.tar -C /home/user01 my_app.ipk
        ```
    * **For a root file system**: Make a `tar` file containing the root file system archive from the MBL build artefacts.

        A symlink to the root file system archive can be found in the build environment at `/path/to/artifacts/machine/<MACHINE>/images/mbl-image-development/images/mbl-image-development-<MACHINE>.tar.xz`

        Where:

        * `/path/to/artifacts` is the output directory specified for all build artefacts. See the [build tutorial](../getting-started/building-an-mbl-image.html) for more information.
        * `<MACHINE>` is the value that was given to the build script for the `--machine` option. See the [build tutorial](../getting-started/building-an-mbl-image.html) to determine which value is suitable for the device in use.

        <span class="notes">**Note:** The file inside the update payload must be named `rootfs.tar.xz` and must be in the tar's root directory, not a subdirectory.</span>

        For example, to create an update payload file at `/tmp/payload.tar` containing a Warp7 root file system image `tar` file with the path `build-mbl/tmp-mbl-glibc/deploy/images/imx7s-warp-mbl/mbl-image-production-imx7s-warp-mbl.tar.xz`, run:
        ```
        user01@dev-machine:~$ tar -cf /tmp/payload.tar -C /path/to/artifacts/machine/imx7s-warp-mbl/images/mbl-image-development/images '--transform=s/.*/rootfs.tar.xz/' --dereference mbl-image-development-imx7s-warp-mbl.tar.xz
        ```
        The `--transform` option renames all files added to the payload to `rootfs.tar.xz` and the `--dereference` option is used so that `tar` adds the actual root file system archive file rather than the symlink to it.
1. Find the device ID in the `mbl-cloud-client` log file at `/var/log/mbl-cloud-client.log`, using the following command on the device's console:

    ```
    root@mbed-linux-os-1234:~# grep -i 'device id' /var/log/mbl-cloud-client.log
    ```

    If you only have one registered device, or if each devices has a been assigned a descriptive name in Portal, you can go to [Device Management Portal](https://portal.mbedcloud.com) > **Device Directory** to find the device ID.

1. For a `rootfs` update, identify which root file system partition is currently active to compare it to the active partition after the update. You can use the [`lsblk` command explained later](#identify-the-active-root-file-system-partition).
1. Change the current working directory to the directory where the manifest tool was initialized. You initialized the manifest tool [when you created the `update_default_resources.c` file](../getting-started/preparing-device-management-sources.html#creating-an-update-resources-file).
1. Run the following command:

    ```
    user01@dev-machine:manifests$ manifest-tool update device --device-id <device-id> --payload <payload-file> --api-key <api-key>
    ```

    Where:

        * `<device-id>` is the device ID.
        * `<payload-file>` is the update payload (`.tar` file).
        * `<api-key>` is the Pelion API key you generated as a prerequisite.

    For a root file system update, the process can take a long time, depending on your file size and network bandwidth.

#### Additional notes

* You can monitor the update payload **download** progress by tailing the `mbl-cloud-client` log file:

    ```
    root@mbed-linux-os-1234:~# tail -f /var/log/mbl-cloud-client.log
    ```

* You can monitor the update payload **installation** progress by tailing:

    * `/var/log/arm_update_activate.log`: for messages about the overall progress of the installation, and messages specific to `rootfs` updates.
    * `/var/log/mbl-app-update-manager-daemon.log`: for messages about application updates.

* For `rootfs` updates, a reboot is automatically initiated to boot into the new firmware. Identify the currently active root file system and verify it was different pre-update. You can use the [`lsblk` command explained later](#identify-the-active-root-file-system-partition).

* The system does not reboot after an application update.

### Identifying the active root file system partition

To verify that a root system update succeeded, you can check which root file system partition is active before and after the update. If the update succeeded, the active partition changes.

To identify the active root file system partition:

```
root@mbed-linux-os-1234:~# lsblk --noheadings --output "MOUNTPOINT,LABEL" | awk '$1=="/" {print $2}'
```

This command prints the label of the block device currently mounted at `/`, which is `rootfs1` if the first root file system partition is active, or `rootfs2` if the second root file system partition is active.

For example, when the first root file system partition is active, you see:

```
root@mbed-linux-os-1234:~# lsblk --noheadings --output "MOUNTPOINT,LABEL" | awk '$1=="/" {print $2}'
rootfs1
root@mbed-linux-os-1234:~#
```

### Identifying the running applications

You can use `runc list` to list all the active applications and their status.

1. The first steps of an application update include stopping and terminating a running instance of the application (if one exists).

    If you perform `runc list` when the application is stopped, you will see:

    ```
    root@mbed-linux-os-1234:/home/app# runc list
    ID                        PID         STATUS      BUNDLE                              CREATED                          OWNER
    user-sample-app-package   0           stopped     /home/app/user-sample-app-package   2018-12-07T08:23:36.742467741Z   root
    ```

    If you perform `runc list` when the application is terminated, the application will not appear on the list.

1. When the application is installed or updated, it starts automatically.

    If you perform `runc list` for a running application, you will see:

    ```
    root@mbed-linux-os-1234:/home/app# runc list
    ID                        PID         STATUS      BUNDLE                              CREATED                          OWNER
    user-sample-app-package   3654        running     /home/app/user-sample-app-package   2018-12-07T08:23:36.742467741Z   root
    ```

    <span class="notes">**Note:** The [Hello World](../getting-started/tutorial-user-application.html) application runs for about 20 seconds. When it finishes, it once again appears as stopped.</span>