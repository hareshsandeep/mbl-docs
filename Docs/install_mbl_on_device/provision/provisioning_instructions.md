## Provisioning instructions

### Prerequisites

* MBL CLI and a developer connection. Please see [installation and developer connection instructions](../develop-apps/setting-up.html).

    <span class="notes">**Note:** If you are running an earlier version (1.x) of MBL CLI, please remove it (and install 2.0).</span>

* We have used [manifest tool](https://github.com/ARMmbed/manifest-tool) 1.4.8 in our testing. You should install the manifest tool in the same virtual environment you created for MBL CLI.
    Install the manifest tool:
    ```
    pip install manifest-tool==1.4.8
    ```

  See the Device Management documentation [for more information on installing the manifest-tool](https://cloud.mbed.com/docs/latest/cloud-requirements/manifest-tutorial.html).

<a href="https://os.mbed.com/account/login/" target="_blank">A Pelion Device Management Account</a>.

### Preliminary steps

#### Mandatory: API key and update authenticity certificate

1. Create an API key for [Pelion Device Management](https://cloud.mbed.com/docs/latest/integrate-web-app/api-keys.html). Be sure to copy the key when prompted - you will need to store it on the device before you begin provisioning.

1. Create an `update_default_resources.c` file with your update authenticity certificate, created with the manifest tool:

    <span class="notes">**Note**: This process is for development only. A production process will be documented when available based on the [factory provisioning process](https://www.pelion.com/docs/device-management/current/provisioning-process/factory-provisioning-process.html).</span>

    1. Create an update resources directory, such as `./update-resources`:

        ```
        mkdir ./update-resources
        ```

    2. Move into the new directory:

        ```
        cd ./update-resources
        ```

     1. Use the [manifest tool](https://www.pelion.com/docs/device-management/current/updating-firmware/manifest-tool.html) to generate an `update_default_resources.c` file. The tool will prompt you to enter some details for a new developer authenticity certificate that it will use in this generation process:

        `manifest-tool init -d <domain> -m <device class>`

        Where:

        * `<domain>` is your company's domain, like `arm.com`.
        * `<device class>` is a unique identifier for the device class. If you're in development (using developer credentials), you can use `dev-device`.

        <span class="warnings">**Warning:** The authenticity certificate generated will be a developer certificate. The Manifest Tool will let you enter a manual expiration value, and will default to 90 if you don't enter one. When the certificate expires, the device will not be updatable. You will need to put a new authenticity certificate on the device, either by [reflashing the device](../first-image/writing-an-image-to-supported-boards.html) or [provisioning the device again](../first-image/provisioning-for-pelion-device-management.html).</span>

        For example:

        ```
        $ manifest-tool init -d arm.com -m dev-device
        A certificate has not been provided to init, and no certificate is provided in .update-certificates/default.der
        Init will now guide you through the creation of a certificate.

        This process will create a self-signed certificate, which is not suitable for production use.

        In the terminology used by certificates, the "subject" means the holder of the private key that matches a certificate.
        In which country is the subject located? UK
        In which state or province is the subject located? Cambridgeshire
        In which city or region is the subject located? Cambridge
        What is the name of the subject organization? Arm
        What is the common name of the subject organization? [arm.com]
        How long (in days) should the certificate be valid? [90]
        [WARNING]: Certificates generated with this tool are self-signed and for testing only
        [WARNING]: This certificate is valid for 90 days. For production,use certificates with at least 10 years validity.
        [INFO] 2019-09-30 10:56:04 - manifesttool.init - Certificate written to .update-certificates/default.der
        [INFO] 2019-09-30 10:56:04 - manifesttool.init - Private key written to .update-certificates/default.key.pem
        [INFO] 2019-09-30 10:56:04 - manifesttool.init - Default settings written to .manifest_tool.json
        [INFO] 2019-09-30 10:56:04 - manifesttool.init - Wrote default resource values to update_default_resources.c
        ```

#### Optional: persistent storage locations

<span class="notes">**Note**: MBL CLI sets up defaults automatically; this manual step is optional, but if you chose to perform it, it should be done before any other provisioning steps.</span>

MBL CLI can fetch and store API keys, firmware update authority certificates and developer certificates from a persistent storage location you can configure. This simplifies the provisioning workflow and makes API authentication easier; MBL CLI will automatically find the API key when it needs to authenticate with the Pelion Device Management service APIs.

You can save your credentials to either a **user** store or a **team** store:

- User Store: Where API keys for each user are stored.

    Default location: `~/.mbl-store/user/` (permissions for this folder are set to `drwx------`). The folder is owned by the `*nix` user who created the store (the store is automatically created on first use).

    MBL CLI can save a single API key in the User Store, which it uses when it needs authentication with the Device Management APIs. If you save another API key, it will overwrite the previously stored key.

- Team Store: Where developer certificates and firmware update authority certificates for your devices are stored; any team member who has access to this folder can provision devices with these credentials (using MBL CLI). If you want to give your team access to this folder, you can set this folder to be shared over a cloud service, or make it discoverable on your network.

    Default location: `~/.mbl-store/team/` (permissions for this folder are set to `drwxrw----`). You should set this store's `user:group` according to the access permissions you have defined for your team.

To specify a storage location, create a config file `~/.mbl-stores.json` with the following contents:

```json
{
    "user": path/to/user/store,
    "team": path/to/team/store
}
```

If, on first use, you do not create your own `~/.mbl-stores.json`, it will be automatically created and populated with the default values.

### Provisioning

To provision your device:

1. Connect the device to your development PC. Follow the instructions to [connect your device](../develop-apps/setting-up.html#setting-up-networking).
2. Run the `select` command. It returns a list of discovered devices; you can select your device by its list index:

    ```bash
    $ mbl-cli select
    Discovering devices. This will take up to 30 seconds.

    1: mbed-linux-os-9999: fe80::21b:63ff:feab:e6a6%enpos623

    $ 1

    ```

3. Store your API key on the device (replacing `<api-key>` with the real API key you saved from the portal).

    ```bash
    mbl-cli save-api-key <api-key>
    ```

    The command validates the key exists in Device Management, then saves the key in the User Store.

4. To provision your device with the developer and update certificates:

    ```bash
    mbl-cli provision-pelion  <cert-name> <update-cert-name> -c -p <update-cert-path>
    ```

    The arguments are:

    | Argument | Info |
    | --- | --- |
    |`<cert-name>`| A name for your developer certificate. |
    |`<update-cert-name>`| A name for your update authenticity certificate, created earlier using the manifest tool.|
    |`-c`| Tells MBL CLI to create a new developer certificate and save it in your Team Store.|
    |`-p`| Tells MBL CLI to parse an existing `update_default_resources.c` file and save it in the Team Store. For example, if your `update_default_resources.c` is located in `path/to/resources/update_default_resources.c` the form of the invocation would be: `-p path/to/resources/update_default_resources.c` |

    MBL CLI injects the certificates into your selected device's secure storage. MBL CLI also saves the certificates, under the given names, in your Team Store for use with other devices.


***

Copyright © 2020 Arm Limited (or its affiliates)
