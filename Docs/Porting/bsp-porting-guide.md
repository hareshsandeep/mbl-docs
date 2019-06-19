# <a name="section-1-0"></a> 1.0 Mbed Linux OS BSP porting guide

## <a name="section-1-1"></a> 1.1 Overview

This document details how to port an existing ARM Cortex-A board support package (BSP) to Mbed Linux OS (MBL), enabling the platform's software stack for security, connection to Pelion Device Management, and firmware update.

Porting a BSP centers on configuring the secure boot software components, so the correct artifacts appear on the right flash partitions for update:

- **Trusted Firmware for Cortex-A (TF-A)**: Use Trusted Firmware in v7A AArch32 and v8A AArch64 secure boot processes. TF-A artifacts include the second-stage boot loader  BL2, and the Firmware Image Package (FIP) containing third-stage boot loaders BL3x and certificates.
- **Open Platform Trusted Execution Environment (OP-TEE)**: This is the OS with trusted applications running in the TrustZone secure world, and is packaged as BL32 in the FIP image.
- **U-Boot**: U-Boot is the Normal world boot loader for loading a Rich OS. This is packaged as BL33 inside the FIP image.
- **Linux kernel**: The Linux kernel is the Normal world Rich OS kernel. The kernel image is packaged with the device tree binaries and initial RAM file system in a Flattened Image Tree (FIT) image.

This document's structure follows the work process:

* Section 1 introduces this guide, including an overview, porting prerequisites, and glossary.

* [Section 2](../develop-mbl/2-0-system-architecture.html) describes the relevant system architecture of [AArch32](../develop-mbl/2-0-system-architecture.html#fig2-2-1) and [AArch64](../develop-mbl/2-0-system-architecture.html#fig2-2-2) secure boot flows, partitioning build artifacts between `BL2`, FIP and FIT images, and the flash partition layout for updating firmware.

* [Section 3](../develop-mbl/3-0-overview-of-mbl-yocto-meta-layers.html) provides a top-down overview of the Yocto meta-layers in an MBL workspace for BSP development, including a [software stack diagram](../develop-mbl/3-0-overview-of-mbl-yocto-meta-layers.html#figure-3.7) showing how recipes from different layers collaborate.

* [Section 4](../develop-mbl/4-0-bsp-recipe-relationships.html) provides an overview of `${MACHINE}.conf`, ATF, OP-TEE, U-Boot and `linux` recipe relationships using a [UML diagram](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0).

* [Section 5](../develop-mbl/5-0-machine-configuration-files.html) discusses, in detail, the MBL `${MACHINE}.conf` and community `${machine}.conf` machine configuration files.

* [Section 6](../develop-mbl/6-0-u-boot.html) discusses the `u-boot*.bb` base recipe and MBL `u-boot*.bbappend` customization.

* [Section 7](../develop-mbl/7-0-linux.html) discusses the `linux*.bb` base recipe and the MBL `linux*.bbappend` customization.

* [Section 8](../develop-mbl/8-0-atf-machine-bb.html) discusses the `atf-${MACHINE}.bb` recipe for building ARM Trusted Firmware.

* [Section 9](../develop-mbl/9-0-example-imx7s-warp-mbl-bsp-recipe-package-relationships.html) provides a concrete example for the WaRP7 target of the `${MACHINE}.conf`, ATF, OP-TEE, U-Boot and `linux` recipe inter-relationships using a [UML diagram](../develop-mbl/9-0-example-imx7s-warp-mbl-bsp-recipe-package-relationships.html#figure-9-1).

* [Section 10](../develop-mbl/10-0-summary-of-bsp-porting-tasks.html) summarizes porting tasks.

* [Section 11](../develop-mbl/11-0-references.html) links to supporting references to this document.

## <a name="section-1-2"></a> 1.2 Prerequisites

MBL uses Yocto, BitBake, `openembedded-core` and third-party meta-layers to compose the development and build workspace.

We recommend reading [Embedded Linux Systems with the Yocto Project][strief-2016] first, then the [Yocto Mega Manual][yocto-mega-manual-latest], as well as the [Yocto Project Board Support Package (BSP) Developer's Guide][yocto-project-board-support-package-bsp-developer-guide-latest].

For porting ATF to your target platform, please consult the [ATF porting guide][atf-doc-plat-porting-guide].

For porting OP-TEE to your target platform, please consult the [OP-TEE documentation][optee-docs].

## <a name="section-1-3"></a> 1.3 Terminology

This section defines terminology used throughout this document.

**Table 1.3: Acronyms and Terminology**
* **REF1:** Term is defined in TF-A fiptool documentation and source code.
* **REF2:** Term is defined in TrustZone documentation.

<a name="Table-1-3"></a>

    Term                Definition
    ----                ----------
    AP                  Application processor
    ATF                 Arm Trusted Firmware
    BL                  Bootloader
    BL1                 First-stage bootloader
    BL2                 Second-stage bootloader. This is based on TF-A running at EL3 when the Memory Management Unit (MMU)
                        is switched off. BL2 loads the FIP image and authenticates FIP content.
    BL31                Third-stage bootloader, part one:
                          - For example, Secure Monitor running in EL1-SW. This stage enables the MMU.
    BL32                Third-stage bootloader, part two:
                          - For example, OP-TEE, the secure world OS. This typically switches to Normal world.
    BL33                Third-stage bootloader, part three:
                          - For example, U-Boot, the Normal world bootloader.
                          - Also referred to as Non-Trusted world firmware (NT-FW).
    DTB                 Device tree binary
    EL                  Execution level
    FIP                 Firmware image package. This is a "simple filesystem" for
                        managing signed bootchain components.
    FIT                 Flattened Image Tree. This is a Linux kernel image container for
                        holding the kernel, kernel DTB and `initramfs`.
    Linux               The runtime Normal world kernel.
    MBL                 Mbed Linux OS
    MMU                 Memory Management Unit
    Normal world        The non-security operating mode as defined in Arm reference documentation.
    NT                  Non-Trusted
    NT-FW               Non-Trusted Firmware binary (REF1)
                          - For example, BL33 U-Boot. Runs at EL2-NW.
    NT-FW-CERT          Non-Trusted Firmware certificate (REF1)
                          - For example, U-Boot content certificate.
    NT-FW-KEY-CERT      Non-Trusted Firmware certificate (REF1)
    NW                  Normal world (REF2)
    OP-TEE              Open Platform Trusted Execution Environment
    Secure world        The high security operating mode as defined in Arm reference document.
    SW                  Secure world (REF2)
    SOC-FW              System-On-Chip Firmware binary (REF1)
    SOC-FW-CERT         System-On-Chip Firmware certificate (REF1)
    SOC-FW-KEY-CERT     System-On-Chip Firmware key certificate (REF1)
    ROT                 Root of Trust
    ROTPK               Root of Trust public key
    ROTPrvK             Root of Trust private key
    TBBR                Trusted Board boot requirements
    TBBR-CLIENT         TBBR specification document
    TB-FW               Trusted Board Firmware binary (REF1)
    TB-FW-CERT          Trusted Board Firmware certificate (REF1)
    TB-FW-KEY-CERT      Trusted Board Firmware key certificate (REF1)
    TF-A                Trusted Firmware for Cortex-A
    TOS-FW              Trusted OS Firmware binary (REF1)
    TOS-FW-CERT         Trusted OS Firmware certificate (REF1)
    TOS-FW-EXTRA1       Trusted OS Firmware Extra-1 binary (REF1)
    TOS-FW-EXTRA2       Trusted OS Firmware Extra-2 binary (REF1)
    TOS-FW-KEY-CERT     Trusted OS firmware key certificate (REF1)
    TRUSTED-KEY-CERT    Trusted Key Certificate.
                          - Contains the trusted world public key.
                          - Contains the non-trusted world public key.
    WIC                 Openembedded Image Creator application.

# <a name="section-2-0"></a> 2.0 System architecture

## <a name="section-2-1"></a> 2.1 Introduction

A summary of the key BSP system architecture:

- **Security:** Trusted Firmware for Cortex-A provides a generic solution for authenticating software components.
- **Firmware Update:** Pelion Device Management Update service is used to update device firmware. This leads to a flash partition layout where trusted firmware, kernel, root file system and applications are independently updatable.
- **Reuse:** Where possible, suitable existing solutions and software are reused to leverage know-how and speed up time to market.

## <a name="section-2-2"></a> 2.2 Boot flow

<a name="fig2-2"></a>

<span class="images">![fig2-2](assets/TWC_before_NWC.png "Figure 2.2")<span><br>**Figure 2.2:** A summary form of the secure boot chain flow</span></span>

[Figure 2.2](#fig2-2) shows the main entities in the secure bootchain sequence: Soc Boot ROM, Trusted Firmware (TF), OP-TEE, U-Boot and Linux kernel:

1. After the power is turned on, the Soc Boot ROM runs. This is the first-stage boot loader (BL1), which is programmed into the chip during manufacture.
1. BL1 authenticates the second-stage bootloader, which is Trusted Firmware for Cortex-A (TF-A). TF-A supplies:
    - The second-stage boot loader BL2.
    - Part 1 of the third-stage boot loader BL31.
1. BL31 runs OP-TEE, also called BL32.
1. BL31 runs the Normal world bootloader, U-Boot (referred to as BL33).
1. U-Boot runs the Linux Kernel.

### <a name="section-2-2-1"></a> 2.2.1 AArch32 boot flow

<a name="fig2-2-1"></a>

<span class="images">![fig2-2-1](assets/LAS16-402_slide_16.png "Figure 2.2.1")<span>**Figure 2.2.1:** Linaro Connect 2016 Presentation LAS16-402 [slide 16][linaro-connect-las16-402-slides] showing the AArch32 secure boot process.</span></span>


[Figure 2.2.1](#fig2-2-1) shows the Cortex-v7A AArch32 generic secure boot process, which is the starting point for discussing secure boot on the WaRP7.

The diagram is divided into four columns, corresponding to the memory type and physical location from which the boot code runs:

1. The first column shows the software components that execute from secure ROM.
1. The second column shows the software components that execute from secure on-chip RAM.
1. The third column shows the software components that execute from secure RAM, which may be on or off the SoC.
1. The fourth column shows the software components that execute from insecure DRAM, which is off-chip.

The boot sequence consists of the following events:

1. BL1 loads BL2.
1. BL1 runs BL2 (in this step and all subsequent steps, running a component is preceded by successful authentication of the component).
1. BL2 loads BL31 (TF-A secure monitor).
1. BL2 loads BL32 (OP-TEE OS).
1. BL2 loads BL33 (U-Boot, the Normal world bootloader).
1. BL2 runs  BL31 (TF-A secure monitor)
1. BL31 runs BL32 (OP-TEE OS). OP-TEE OS modifies the kernel device tree to communicate shared information between OP-TEE OS and the kernel, for example the address and size of the shared memory buffer that is used OP-TEE OS and the kernel.
1. BL32 runs U-Boot (change from SW to NW).
1. BL33 (u-boot) runs kernel.

The secure boot chain process is now complete.

### <a name="section-2-2-2"></a> 2.2.2 AArch64 boot flow

<a name="fig2-2-2"></a>

<span class="images">![fig2-2-2](assets/LAS16-402_slide_15.png "Figure 2.2.2")<span>**Figure 2.2.2:** Linaro Connect 2016 Presentation LAS16-402 [slide 15][linaro-connect-las16-402-slides] showing AArch64 secure boot process.</span></span>

[Figure 2.2.2](#fig2-2-2) shows the Cortex-v8A AArch64 generic secure boot process, which is the starting point for discussing the Raspberry Pi 3 and NXP IMX8 Mini secure boot.

For steps 1-6, the boot flow for AArch64 is the same as the AArch32 boot flow described in the previous section. Thereafter, the boot flow differs slightly:

7. BL31 runs BL32, and then blocks waiting for BL32 to complete initialization.
8. BL32 (Secure Payload, OP-TEE) runs and initializes.
9. BL31 (SoC AP Firmware, Secure Monitor) resumes and runs BL33 (Normal World Firmware, U-Boot). BL31 continues to run in the system.
10. BL33 orchestrates the loading and running of the Rich OS.

The secure boot chain process is now complete.

See the [Basic Signing Flow document][basic-signing-flow] for a more detailed description of the AArch64 secure boot flow.

## <a name="section-2-3"></a> 2.3 Partitioning software components into FIP/FIT images

<a name="fig2-3"></a>

<span class="images">![fig2-3](assets/Image_signing_flow.png "Figure 2.3")<span>**Figure 2.3:** partitioning of software components.</span></span>

[Figure 2.3](#fig2-3) shows the factoring of software components into five binary images:

1. **SoC Compatible Image:** This image contains the TF-A generated BL2 and the ROTPK, and is signed.
1. **FIP Image:** This image is the TF-A fiptool-generated FIP image, and contains many [TBBR-CLIENT-defined key and content certificates](#ref-tbbr-client), as well as the BL3x bootchain components.

    The FIP image contains the following components:

    1. TRUSTED-KEY-CERT.
    1. SOC-FW-KEY-CERT1.
    1. TOS-FW-KEY-CERT.
    1. NT-FW-KEY-CERT.
    1. SOC-FW-CERT.
    1. BL31 (TF-A).
    1. TOS-FW-CERT.
    1. BL32 (OP-TEE).
    1. BL33 (u-boot).
    1. U-Boot device tree containing the FIT verification public key.
1. **FIT Image:** Use `u-boot-mkimage` to create the FIT image, which contains:
    1. Linux kernel.
    1. Linux kernel device tree.
    1. `boot.scr`. This is a compiled version of the U-Boot boot script.
    1. The initramfs image.
    1. [The Verity public key.][android-verified-boot]
    1. A configuration block.
1. **Rootfs Partition:** This image contains the root filesystem.
1. **Rootfs_hash Partition:** This image contains the Verity hash tree.

For more information, please refer to the [Trusted Board Boot Requirements CLIENT document](#ref-tbbr-client).

## <a name="section-2-4"></a> 2.4 Flash partition layout

<a name="fig2-4"></a>

<span class="images">![fig2-4](assets/flash_partition_layout.png "Figure 2.4")<span>**Figure 2.4:** The flash partition layout to support update has 2 banks of images.</span></span>

[Figure 2.4](#fig2-4) shows the flash partition layout, and the function of each partition is described in the table below.

| Partition | Usage |
| --- | --- |
| Bank/Update state | This is a raw partition that is accessible by all bootloaders and the normal device firmware. It holds the non-volatile state that reflects the active bank and whether an update is in progress. It is important that updates to state are robust to power failure. |
| BL2 | This is a raw partition that holds the TF-A BL2 bootloader. BL2 cannot be updated using the normal firmware update process. |
| BL3 FIP Image 1 & 2 | Two partitions to hold two versions of the BL31 boot loader and associated components contained within a signed FIP image. |
| Boot FIT Image 1 & 2 | Two partitions to hold two versions of the boot partition. Contains the Linux kernel and device tree. |
| Rootfs 1 & 2 | Two partitions for the read-only rootfs and the associated dm-verity hash tree. |
| Rootfs_Hash 1 & 2 | Partitions for the dm-verity hash trees corresponding to rootfs 1 & 2. |
| Config 1 & 2 | Non-volatile configuration data is saved to the active config partition. Two partitions are used, to allow an update to modify configuration data while maintaining a fallback to the old data if an update fails. In most cases, configuration data is just copied between banks during an update. |
| Factory Config | A single partition for configuration data written during the manufacturing process. When manufacturing is complete, the partition is modified to being read-only. |
| Log | A single partition for log files. |
| Scratch | `/scratch` directory mounted to this partition. Used for saving potentially large temporary files such as downloaded firmware files. Note that `/tmp` is used in a similar manner, but is mapped to the RAM file system where there is less file storage available. |
| Home | `/home` directory mounted to this partition. Used for user space application storage. |


**Table 2.4: Flash partition functional description.**

The flash partitions for the following software components are banked (there are two instances present in the system):

- BL3 FIP Image.
- Boot FIT Image.
- Rootfs.
- Rootfs_Hash.
- Config.

The flash partitions are banked to support the update service: One partition is the active (running partition), while the other is the inactive (non-running) partition. An update writes a new image to the inactive partition, then changes the bank/update state to bring the new image into service (the inactive bank becomes active, and the active bank becomes inactive).

# <a name="section-3-0"></a> 3.0 Overview of MBL Yocto meta-layers

## <a name="section-3-1"></a> 3.1 Types of Yocto meta-layers

The MBL workspace contains Yocto community and MBL meta-layers needed to build MBL images. The Yocto project classifies layers into one of three types:

- **BSP layers**, which contain the machine configuration file for a target platform, or metadata relating to target-specific board support packages (for example, `meta-raspberrypi`).
- **Distro layers**, which contain the configuration file (`mbl.conf`) for the distribution (for example, `meta-mbl-distro`).
- **General purpose layers**, which contain metadata and recipes for applications (for example, `meta-mbl-apps`).

MBL introduces the additional **staging layer**. The staging layer provides a logical place where MBL-originated `.bb` and `.bbappend` recipes relating to a community layer can be stored prior to upstreaming, or if the metadata cannot be upstreamed, maintained for the MBL distribution.

To introduce the community and MBL meta-layers used in MBL, [Section 3.2](#section-3-2) offers a concrete example of layers used in the `raspberrypi3-mbl` workspace. The distribution and general purpose layers are only briefly mentioned, whereas BSP layers are described in more detail.

The subsequent sections describe the BSP meta-layers used in the remaining target platforms: [Section 3.3](#section-3-3) for `imx7d-pico-mbl`, [Section 3.4](#section-3-4) for `imx7s-warp-mbl` and [Section 3.5](#section-3-5) for `imx8mmevk-mbl`. For all three platforms, the distribution and general purpose layers are the same as `raspberrypi3-mbl`.

## <a name="section-3-2"></a> 3.2 Layers for `raspberrypi3-mbl`

Once you've created an MBL workspace and initialized the environment, you can list the `bblayers*.conf` configured meta-layers using `bitbake-layers show-layers`.

[Table 3.2.1](#Table-3-2-1) shows the command output for `MACHINE=raspberrypi3-mbl`:

<a name="Table-3-2-1"></a>

| Layer | Path | Priority |
| --- | --- | --- |
| meta-mbl-distro |  <ws>/layers/meta-mbl/meta-mbl-distro | 10 |
| meta-mbl-apps | <ws>/layers/meta-mbl/meta-mbl-apps | 7 |
| meta-filesystems | <ws>/layers/meta-openembedded/meta-filesystems | 6 |
| meta-networking | <ws>/layers/meta-openembedded/meta-networking | 5 |
| meta-oe | <ws>/layers/meta-openembedded/meta-oe | 6 |
| meta-python | <ws>/layers/meta-openembedded/meta-python | 7 |
| meta-virtualization-mbl | <ws>/layers/meta-mbl/meta-virtualization-mbl | 9 |
| meta-virtualization | <ws>/layers/meta-virtualization | 8 |
| meta-mbl-bsp-common | <ws>/layers/meta-mbl/meta-mbl-bsp-common | 10 |
| meta-raspberrypi-mbl | <ws>/layers/meta-mbl/meta-raspberrypi-mbl | 11 |
| meta-raspberrypi | <ws>/layers/meta-raspberrypi | 9 |
| meta-optee | <ws>/layers/meta-mbl/meta-linaro-mbl/meta-optee | 9 |
| meta-optee | <ws>/layers/meta-linaro/meta-optee | 8 |
| meta | <ws>/layers/meta-mbl/openembedded-core-mbl/meta | 6 |
| meta | <ws>/layers/openembedded-core/meta | 5 |

**Table 3.2.1:** Output of `bitbake-layers show-layers` for `MACHINE=raspberrypi3-mbl` in table form.

Note that the command output has been slightly modified for presentation purposes (for example, the full path to the MBL workspace path has been shortened to `<ws>`).

- The first column shows the meta-layer name, which is also the name of the workspace directory containing the layer.
- The second column shows the path to the meta-layer. The layout of the layers is more clearly explained by the directory hierarchy shown in [Figure 3.2](#Figure-3-2).
- The third column shows the priority of the layer, which controls the BitBake layer processing order. Layers with a higher priority number are processed after lower numbers, so the settings in the higher priority number layer take precedence.

The MBL workspace directory structure in [Figure 3.2](#Figure-3-2) shows:

- The community often stores multiple meta-layers in a single repository. For example, the `meta-openembedded` repository contains the layers `meta-filesystems`, `meta-networking`, `meta-oe` and `meta-python`. In this case, the `meta-openembedded` repository name appears as a subdirectory of `layers`, and the meta-layers are subdirectories of `meta-openembedded`.

  The point to observe here is that if multiple layers are provided by a repository, then both the repository name and the layer name are preserved in the workspace (other examples include `meta-linaro/meta-optee` and `openembedded-core/meta`). Otherwise, the meta-layer appears directly under `layers` (for example, `meta-raspberrypi` and `meta-virtualisation`).

- Like the community, MBL stores multiple layers in the `meta-mbl` repository. The workspace `layers/meta-mbl` directory stores multiple layers.

- The `meta-mbl` repository stores new layers, for example, `meta-mbl-apps`, `meta-mbl-bsp-common` and `meta-mbl-distro`. The new MBL layers are reusable components of related meta-data. A third-party distribution can use MBL secure boot by reusing `meta-mbl-bsp-common`.

- New MBL layers have `meta-mbl` at the start of the layer name.

- The `meta-mbl` repository stores staging layers for customizations of community recipes (such as `.bbappend` recipes).

- Staging layers follow the naming convention of appending `-mbl` to the community repository. For example, `meta-linaro-mbl/meta-optee`, `openembedded-core-mbl/meta`, `meta-raspberrypi-mbl`, and `meta-virtualization-mbl`.

- In the staging layers configuration file (`layers.conf`) the `BBFILE_COLLECTIONS` variable should append `-mbl` to the upstream layer original value. For example:
    - For `meta-linaro-mbl/meta-optee/conf/layer.conf`: `BBFILE_COLLECTIONS = "meta-optee-mbl"`
    - For `openembedded-core-mbl/meta/conf/layer.conf`: `BBFILE_COLLECTIONS = "core-mbl"`
    - For `meta-raspberrypi-mbl/conf/layer.conf`: `BBFILE_COLLECTIONS = "raspberrypi-mbl"`
    - For `meta-virtualization-mbl/conf/layer.conf`: `BBFILE_COLLECTIONS = "virtualization-layer-mbl"`

<a name="Figure-3-2"></a>

```
    <mbl_workspace_root_path>
    └── layers                                  // Directory containing meta-layers at leaf nodes.
        ├── meta-linaro                         // Community repo name holding multiple meta-layers.
        │   └── meta-optee                      // Community meta-layer for Trusted Exec. Env.
        ├── meta-mbl                            // MBL repo name holding multiple meta-layers.
        │   ├── meta-linaro-mbl                 // MBL staging directory for meta-linaro.
        │   │   └── meta-optee                  // MBL staging layer for meta-optee customizations.
        │   ├── meta-mbl-apps                   // MBL layer for MBL applications.
        │   ├── meta-mbl-bsp-common             // MBL layer for common BSP recipes & meta-data.
        │   ├── meta-mbl-distro                 // MBL distribution layer.
        │   ├── meta-raspberrypi-mbl            // MBL staging layer for meta-raspberrypi `*.bbappend`.
        │   ├── meta-virtualization-mbl         // MBL staging layer for meta-virtualization `*.bbappend`.
        │   └── openembedded-core-mbl           // MBL staging directory for openembedded-core.
        │       └── meta                        // MBL staging layer for openembedded-core/meta
        ├── meta-openembedded                   // Community repo name holding multiple meta-layers.
        │   ├── meta-filesystems                // Community meta-layer for filesystems.
        │   ├── meta-networking                 // Community meta-layer for networking.
        │   ├── meta-oe                         // Community meta-layer for Open Embedded.
        │   └── meta-python                     // Community meta-layer for Python.
        ├── meta-raspberrypi                    // Community meta-layer for Raspberry Pi BSP.
        ├── meta-virtualization                 // Community meta-layer for virtualization.
        └── openembedded-core                   // Community repo name holding multiple meta-layers.
            └── meta                            // Community meta-layer for building Linux distributions.
```

**Figure 3.2:** Workspace `layer` directory hierarchy representation showing `raspberrypi3-mbl` meta-layers.

For the community layer `meta-raspberrypi`, the `meta-mbl` repository contains the MBL staging layer `meta-raspberrypi-mbl` for `.bbappend` customizations of
`meta-raspberrypi *.bb` recipes. Because `meta-raspberrypi-mbl` contains the `raspberrypi3-mbl.conf` machine configuration file, it is also a BSP layer.
`raspberrypi3-mbl.conf` cannot be upstreamed to `meta-raspberrypi`, and therefore has to be maintained independently.

[Table 3.2.2](#Table-3-2-2) summarizes the layers that appear in the MBL workspace.

<a name="Table-3-2-2"></a>

| Layer | Type | Source | Description |
| --- | --- | --- | --- |
| openembedded-core/meta                    | General   | Community | Openembedded core recipe library support for building images. |
| openmebedded-core-mbl/meta                | General   | MBL       | MBL staging layer for `openembedded-core/meta` customizations. |
| meta-filesystems                          | General   | Community | Filesystem subsystems meta-layer. |
| meta-freescale                            | BSP       | Community | Freescale NXP-maintained BSP layer for i.MX8 target containing `imx8mmevk.conf`. |
| meta-freescale-mbl                        | BSP       | MBL       | MBL BSP staging layer containing `imx8mmevk-mbl.conf`, `u-boot*.bbappend` and `linux*.bbappend` recipe customizations. |
| meta-freescale-3rdparty                   | BSP       | Community | The Freescale NXP community has established this low-friction alternative for upstreaming third party originated recipes. i.MX7 targets including `imx7s-warp.conf` and `imx7d-pico.conf` are hosted in this layer. |
| meta-freescale-3rdparty-mbl               | BSP       | MBL       | MBL BSP staging layer containing `imx7*-mbl.conf`, `u-boot*.bbappend` and `linux*.bbappend` recipe customizations. |
| meta-fsl-bsp-release-mbl/imx/meta-bsp     | BSP       | MBL       | MBL BSP staging layer containing Qualcomm qca9377 firmware installation script and imx8 firmware blobs recipe customization. |
| meta-fsl-bsp-release/imx/meta-bsp         | BSP       | Community | BSP layer containing Qualcomm qca9377 firmware and kernel module recipes and imx8 firmware blobs used by some NXP targets and qcom qca9377 firmware and kernel modules. |
| meta-linaro/meta-optee                    | BSP       | Community | Linaro-provided layer for OP-TEE |
| meta-linaro-mbl/meta-optee                | BSP       | MBL       | MBL staging layer for `meta-optee` customizations or related meta-data. |
| meta-mbl-apps                             | General   | MBL       | MBL applications such as `mbl-cloud-client`. |
| meta-mbl-bsp-common                       | BSP       | MBL       | MBL layer for BSP meta-data commonly used by more than one target BSP. |
| meta-mbl-distro                           | Distro    | MBL       | MBL distribution layer including image recipes containing `mbl.conf`, `mbl-image*.bb` recipes and `*.wks` files. |
| meta-networking                           | General   | Community | Networking subsystems meta-layer. |
| meta-oe                                   | General   | Community | Open Embedded layer for distribution tools and applications. |
| meta-python                               | General   | Community | Layer to build the Python runtime for the target. |
| meta-raspberrypi                          | BSP       | Community | Raspberry Pi provided BSP layer containing `raspberrypi3.conf`. |
| meta-raspberrypi-mbl                      | BSP       | MBL       | MBL staging layer for `meta-raspberrypi` customizations. |
| meta-virtualization                       | General   | Community | Layer to provide support for constructing OE-based virtualized solutions. |
| meta-virtualization-mbl                   | General   | MBL       | MBL staging layer for Docker virtualization customizations. |

**Table 3.2.2:** All the meta-layers in the MBL workspace.

Note that an MBL workspace contains all of the meta-layers listed in Table 3.2.2, but the `bblayers*.conf` files configure BitBake to only use the meta-layers needed for the current target and ignore the rest. This is achieved by:
- `bblayers.conf` only specifying the layers common to all targets.
- `bblayers.conf` including a target-specific file `bblayers_${MACHINE}.conf`, which specifies the target-specific layers.

## <a name="section-3-3"></a> 3.3 BSP meta-layers for `imx7d-pico-mbl`

[Table 3.3.1](#Table-3-3-1) shows the BSP layers for `imx7d-pico-mbl` configured in `bblayers_imx7d-pico-mbl.conf`. The full set of layers used by `imx7d-pico-mbl` is the set of layers obtained by replacing the `meta-raspberrypi*` BSP layers in [Table 3.2.1](#Table-3-2-1) with the BSP layers in [Table 3.3.1](#Table-3-3-1) below.

Refer to [Section 3.2](#section-3-2) for details of the layers.

<a name="Table-3-3-1"></a>

| Layer | Path | Priority |
| --- | --- | --- |
| meta-freescale-mbl | <ws>/layers/meta-mbl/meta-freescale-mbl | 11 |
| meta-freescale | <ws>/layers/meta-freescale | 5 |
| meta-freescale-3rdparty-mbl | <ws>/layers/meta-mbl/meta-freescale-3rdparty-mbl | 11 |
| meta-freescale-3rdparty | <ws>/layers/meta-freescale-3rdparty | 4 |
| meta-bsp | <ws>/layers/meta-mbl/meta-fsl-bsp-release-mbl/imx/meta-bsp  | 9 |
| meta-bsp | <ws>/layers/meta-fsl-bsp-release/imx/meta-bsp | 8 |

**Table 3.3.1:** The BSP layers output from `bitbake-layers show-layers` for `MACHINE=imx7d-pico-mbl` in table form.

## <a name="section-3-4"></a> 3.4 BSP meta-layers for `imx7s-warp-mbl`

[Table 3.4.1](#Table-3-4-1) shows the BSP layers for `imx7s-warp-mbl` configured in `bblayers_imx7s-warp-mbl.conf`. The full set of layers used by `imx7s-warp-mbl` is the set of layers obtained by replacing the `meta-raspberrypi*` BSP layers in [Table 3.2.1](#Table-3-2-1) with the BSP layers in [Table 3.4.1](#Table-3-4-1) below.

Refer to [Section 3.2](#section-3-2) for details of the layers.

<a name="Table-3-4-1"></a>

| Layer | Path | Priority |
| --- | --- | --- |
| meta-freescale-mbl | <ws>/layers/meta-mbl/meta-freescale-mbl | 11 |
| meta-freescale | <ws>/layers/meta-freescale  | 5 |
| meta-freescale-3rdparty-mbl | <ws>/layers/meta-mbl/meta-freescale-3rdparty-mbl | 11 |
| meta-freescale-3rdparty | <ws>/layers/meta-freescale-3rdparty | 4 |

**Table 3.4.1:** The BSP layers output from `bitbake-layers show-layers` for `MACHINE=imx7s-warp-mbl` in table form.

## <a name="section-3-5"></a> 3.5 BSP meta-layers for `imx8mmevk-mbl`

[Table 3.5.1](#Table-3-5-1) shows the BSP layers for `imx8mmevk-mbl` configured in `bblayers_imx8mmevk-mbl.conf`. The full set of layers used by `imx8mmevk-mbl` is the set of layers obtained by replacing the `meta-raspberrypi*` BSP layers in [Table 3.2.1](#Table-3-2-1) with those in [Table 3.5.1](#Table-3-5-1) below.

Refer to [Section 3.2](#section-3-2) for details of the layers.

<a name="Table-3-5-1"></a>

| Layer | Path | Priority |
| --- | --- | --- |
| meta-freescale-mbl | <ws>/layers/meta-mbl/meta-freescale-mbl | 11 |
| meta-freescale | <ws>/layers/meta-freescale | 5 |
| meta-bsp | <ws>/layers/meta-mbl/meta-fsl-bsp-release-mbl/imx/meta-bsp | 9 |
| meta-bsp | <ws>/layers/meta-fsl-bsp-release/imx/meta-bsp | 8 |

**Table 3.5.1:** The BSP layers output from `bitbake-layers show-layers` for `MACHINE=imx8mmevk-mbl` in table form.

## <a name="section-3-6"></a> 3.6 Example machine configuration files

In this document, BSP layers are often referred to as `meta-[soc-vendor]` and `meta-[soc-vendor]-mbl`, respectively, when the discussion is applicable to all targets. Specific layer reference examples are:

- `meta-raspberrypi` and `meta-raspberrypi-mbl`, BSP layers for Raspberry Pi.
- `meta-freescale` and `meta-freescale-mbl`, BSP layers for the Freescale NXP i.MX8 Mini.

[Table 3.6](#Table-3-6) shows the relationship between the target machine configuration files and the containing meta-layers:

- The first column defines the MACHINE identifier.
- The second column provides the name of the `${MACHINE}.conf` file contained in the `meta-[soc-vendor]-mbl` MBL staging layer.
- The third column provides the name of the `${machine}.conf` file contained in the `meta-[soc-vendor]` community layer.
- The forth column provides the meta-layers that hold the machine configuration files. `meta-freescale(-3rdparty)(-mbl)` denotes four layers:

    - `meta-freescale`, `meta-[soc-vendor]` community layer.
    - `meta-freescale-mbl`, MBL staging layer.
    - `meta-freescale-3rdparty`, `meta-[soc-vendor]` community layer.
    - `meta-freescale-3rdparty-mbl`, MBL staging layer.

<a name="Table-3-6"></a>

| MACHINE | `${MACHINE}.conf` | `${machine}.conf` | meta-layer(s) |
| --- | --- | --- | --- |
| `imx7s-warp-mbl` | `imx7s-warp-mbl.conf` | `imx7s-warp.conf` | `meta-freescale(-3rdparty)(-mbl)` |
| `imx8mmevk-mbl` | `imx8mmevk-mbl.conf` | `imx8mmevk.conf` | `meta-freescale(-mbl)` |
| `raspberrypi3-mbl` | `raspberrypi3-mbl.conf` | `raspberrypi3.conf` | `meta-raspberrypi(-mbl)` |
| `imx7d-pico-mbl` | `imx7d-pico-mbl.conf` | `imx7d-pico.conf` | `meta-freescale(-3rdparty)(-mbl)` |

**Table 3.6:** `${MACHINE}.conf`, `${machine}.conf` and the associated meta-layers.

## <a name="section-3-7"></a> 3.7 Yocto BSP recipe software architecture

This section gives a top-down overview of the MBL Yocto meta-layers and the relationships between recipes and configuration files.

<a name="figure-3.7"></a>

<span class="images">![figure-3.7](assets/mbl_yocto_workspace_layers.png)<span>**Figure 3.7:** The Yocto meta-layers relevant for BSP development. `meta-mbl` repo entities are shown in blue, `meta-[soc-vendor]` in green, `meta-optee` in orange and `openembedded-core` in yellow.</span></span>

The MBL development workspace is composed of the Yocto layers related to BSP development as shown in [Figure 3.7](#figure-3.7).

Each layer is shown horizontally, containing a number of recipe packages and configuration files. Beginning with the top layer and working downwards:

- **`meta-mbl-distro`**. The distribution layer provides the WIC kickstart image layout files `${MACHINE}.wks`.
- **`meta-[soc-vendor]-mbl`**. The MBL staging layer provides:
    - The BSP customization for specific target platforms by defining `${MACHINE}.conf` files.
    - The MBL `u-boot*.bbappend` customization recipes to build U-Boot.
    - The MBL `linux*.bbappend`  customization recipes to build Linux.
    - The `atf-${MACHINE}.bb` recipe to build ATF. This includes `atf.inc` from the `meta-mbl-bsp-common` layer.
- **`meta-[soc-vendor]`**. The community layer provides:
    - The BSP support for specific target platforms. That is, it defines `${machine}.conf` files.
    - The `u-boot*.bb` base recipes and customizations using the `u-boot*.bbappend` recipes.
    - The `linux*.bb` base recipes and customizations using the`linux*.bbappend` recipes.
- **`meta-mbl-bsp-common`**. This MBL layer contains the generic ATF recipe support `atf.inc` which is used by the target-specific `atf-${MACHINE}.bb` recipe.
- **`meta-linaro-mbl/meta-optee`**. This MBL staging layer provides the `optee*.bbappend` customization recipes.
- **`meta-optee`**. The community layer provides:
    - `optee-os.bb` for building the OP-TEE OS.
    - `optee-client.bb` for building the trusted execution client library for the Linux kernel.
    - `optee-test.bb` for building the OP-TEE test framework and tests.
- **`openembedded-core-mbl/meta`**. This MBL staging layer provides:
    - `mbl-fitimage.bbclass`, a reusable class used to generate the kernel FIT packaging. See [Section 7.4](../develop-mbl/7-0-linux.html#7-4-kernel-fitimage-bbclass-and-mbl-fitimage-bbclass) for details.
- **`openembedded-core`**. This layer contains a library of recipes and classes supporting the creation of Linux distributions:
    - `u-boot.inc.`. This `include` file contains the bulk of the symbol definitions and recipe functions for building the U-Boot bootloader. It's included into the `u-boot_${PV}.bb` recipe.
    - `u-boot-sign.bbclass`. The class that orchestrates verified boot signing of FIT images.
    - `u-boot_${PV}.bb`. The top level boilerplate recipe for building the U-Boot bootloader. The package version variable `${PV}` expands to give `u-boot_2018.11.bb`, for example.
    - `u-boot-tools_${PV}.bb`. A recipe for building the U-Boot `mkimage` tool, which can, for example, create and sign FIT images.

      You can use the recipe to build either `mkimage` host or target versions:

    - `u-boot-fw_utils_{PV}.bb`. A recipe for building the U-Boot `fw_printenv/fw_setenv/etc` firmware tools for managing the U-Boot environment.

      The recipe can build either host or target binaries:

    - `u-boot-common_${PV}.inc`. This `include` file contains common symbol definitions used by multiple `u-boot*` recipes. It is included in the `u-boot_${PV}.bb` recipe.
    - `kernel-fitimge.bbclass`. See [Section 7.4](../develop-mbl/7-0-linux.html#7-4-kernel-fitimage-bbclass-and-mbl-fitimage-bbclass) for details.
    - `kernel-devicetree.bbclass`. See [Section 7.3](../develop-mbl/7-0-linux.html#7-3-kernel-bbclass-openembedded-core-support) for details.
    - `kernel-uimage.bbclass`. See [Section 7.3](../develop-mbl/7-0-linux.html#7-3-kernel-bbclass-openembedded-core-support) for details.
    - `kernel-module-split.bbclass`. See [Section 7.3](../develop-mbl/7-0-linux.html#7-3-kernel-bbclass-openembedded-core-support) for details.
    - `kernel-uboot.bbclass`. See [Section 7.4](../develop-mbl/7-0-linux.html#7-4-kernel-fitimage-bbclass-and-mbl-fitimage-bbclass) for details.

# <a name="section-4-0"></a> 4.0 BSP recipe relationships

This section describes the main BSP recipe relationships using a UML diagram. The discussion is applicable to all targets.

<a name="figure-4-0"></a>

![figure-4-0](assets/mbl_machine_config_uml_summary.png "Figure 4.0")

**Figure 4.0: The figure shows important configuration and recipe file relationships. `meta-mbl` repo entities are shown in blue, `meta-[soc-vendor]`
  in green, `meta-optee` in orange and `openembedded-core` in yellow.**

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) illustrates the key relationships between important recipe and configuration packages in a UML diagram.
The model captures an abstract understanding of how the different recipe components fit together to control the MBL build for any target.

Note that an entity's color indicates the layer in which it resides and follows the same color coding used in [Figure 3.7](#figure-3.7).

The `${MACHINE}.conf` is the top level control file specifying how the key boot components (ATF, OP-TEE, U-Boot and Linux) form a working bootchain. It includes the `${machine}.conf` supplied by the `meta-[soc-vendor]` BSP layer, which in turn includes `[soc-family].inc`. For more information on `${MACHINE}.conf`, `${machine}.conf` and `[soc-family].inc`, see [Section 5.0](../develop-mbl/5-0-machine-configuration-files.html).

The `[soc-family].inc` specifies the U-Boot recipe by setting `PREFERRED_PROVIDER_virtual/bootloader = u-boot-XXXX`. The `u-boot*.bb` base recipe controls building U-Boot as the bootloader, subject to machine configuration file settings. For more information on `u-boot*` processing, see [Section 6.0](../develop-mbl/6-0-u-boot.html).

The `[soc-family].inc` specifies the Linux kernel recipe by setting `PREFERRED_PROVIDER_virtual/kernel = linux-XXXX`. The `linux*.bb` base recipe controls building `linux` as the kernel, subject to machine configuration file settings. For more information on `linux*` processing, see [Section 7.0](../develop-mbl/7-0-linux.html).

The `atf-${MACHINE}.bb` is the target specific ATF recipe that controls how the ATF components of the bootchain are built and packaged. `atf-${MACHINE}.bb` uses `atf.inc`, which encapsulates the generic ATF processing common to all targets. `atf.inc` uses `optee-os.bb`, which builds the OP-TEE component. For more information on `atf-${MACHINE}.bb`, `atf.inc`  and `optee-os.bb` processing, see [Section 8.0](../develop-mbl/8-0-atf-machine-bb.html).

# <a name="section-5-0"></a> 5.0 Machine configuration files

This section describes the `${MACHINE}.conf`, `${machine}.conf` and `[soc-family].inc` entities in the BSP recipe relationship UML diagram ([Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0)). The discussion is applicable to all targets.

## <a name="section-5-1"></a> 5.1 `${MACHINE}.conf`: The top level BSP control file

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) illustrates the `${MACHINE}.conf` machine configuration file using a UML class entity with symbols.
The MBL `meta-[soc-vendor]-mbl ${MACHINE}.conf` file includes the community `meta-[soc-vendor] ${machine}.conf` and customizes key symbols to specify how ATF, OP-TEE, U-Boot and `linux` will be built and configured. MBL uses `${MACHINE}.conf` to override and modify the configuration specified configuration in `${machine}.conf`.

The key symbols modified in `${MACHINE}.conf` are as follows:
- `PREFERRED_PROVIDER_virtual/atf = "atf-${MACHINE}"`. This symbol in `${MACHINE}.conf` specifies which recipe to use to build ATF. The recipe packages bootchain artifacts into the FIP image as specified in [Section 2.3](../develop-mbl/2-0-system-architecture.html#2-3-partitioning-software-components-into-fip-fit-images).
- `KERNEL_CLASSES = "mbl-fitimage"`. This symbol changes the `kernel.bbclass` processing to inherit the `mbl-fitimage.bbclass`, which
  packages the kernel in a FIT image as specified in [Section 2.3](../develop-mbl/2-0-system-architecture.html#2-3-partitioning-software-components-into-fip-fit-images).
- `KERNEL_IMAGETYPE = "fitImage"`. This symbol customizes `kernel.bbclass` processing to generate a FIT image rather than a zImage, for example.
- `KERNEL_DEVICETREE = "XXX"`. This symbol definition is used to specify additional device trees that can be included in the FIT image.
- `UBOOT_ENTRYPOINT = "0xabcdefab"`. This symbol specifies the U-Boot entry point called by OP-TEE, for example.
- `UBOOT_DTB_LOADADDRESS = "0xabcdefab"`. This symbol specifies the memory address where the U-Boot DTB is loaded into memory.
- `UBOOT_SIGN_ENABLE = "1"`. This symbol enables FIT image signing of subcomponents by `u-boot-mkimage`.
- `WKS_FILE = "${MACHINE}.wks"`. This symbol specifies the WIC kickstart file defining the target partition layout.
   See "Creating Partitioned Images Using Wic" in [Yocto Mega Manual][yocto-mega-manual-latest] and [Section 2.4](../develop-mbl/2-0-system-architecture.html#2-4-flash-partition-layout) for more details.

[Section 2.3 Partitioning software components into FIP/FIT image](../develop-mbl/2-0-system-architecture.html#2-3-partitioning-software-components-into-fip-fit-images) specifies that the Linux kernel image is packaged into a FIT image so the kernel FIT image can be written to a [dedicated partition](../develop-mbl/2-0-system-architecture.html#2-4-flash-partition-layout) and independently updated. FIT image generation is achieved using the `linux*`, `kernel.bbclass`, `mbl-fitimage.bbclass` and `kernel-fitimage.bbclass` entities shown in [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html),
and by setting the symbols `KERNEL_CLASSES` and `KERNEL_IMAGETYPE`. See [Section 7.3](../develop-mbl/7-0-linux.html#7-3-kernel-bbclass-openembedded-core-support) and [Section 7.4](../develop-mbl/7-0-linux.html#7-4-kernel-fitimage-bbclass-and-mbl-fitimage-bbclass) for more details.

See [Section 9.1](../develop-mbl/9-0-example-imx7s-warp-mbl-bsp-recipe-package-relationships.html#9-1-example-imx7s-warp-mbl-recipe-package-uml-diagram) for details on the `${MACHINE}.conf` file for `imx7s-warp-mbl`.

## <a name="section-5-2"></a> 5.2 `${machine}.conf`: The community BSP control file

The `meta-[soc-vendor]` machine configuration files `${machine}.conf` orchestrate U-Boot and kernel creation using virtual providers (see the section "Using Virtual Providers" in the [Yocto Mega Manual][yocto-mega-manual-latest]). Virtual providers allow the selection of a specific package recipe from among several providers. For example, consider the case of two `u-boot*` recipes each providing the same package functionality
by declaring they provide the `virtual/bootloader` symbolic package name:
- `u-boot-fslc.bb` declares its ability to build a boot loader by specifying the virtual provider directive `PROVIDES="virtual/bootloader"`.
- `u-boot-imx.bb` declares the virtual provider directive `PROVIDES="virtual/bootloader"`.

A `${machine}.conf` (by including `[soc-family].inc`) selects a specific boot loader package recipe by setting the `PREFERRED_PROVIDER_virtual/bootloader`
symbol to the actual recipe (package) name:

    PREFERRED_PROVIDER_virtual/bootloader="u-boot-fslc"

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) shows it is the `[soc-family].inc` recipe included by `${machine}.conf`
that specifies the virtual providers for the U-Boot and kernel components:
`[soc-family].inc` is an include file containing target SoC symbol definitions common to a family of processors, and may be used in
more than one `${machine}.conf`. For example:
- `[soc-family].inc` specifies the U-Boot recipe by setting `PREFERRED_PROVIDER_virtual/bootloader = u-boot-XXXX`.
- `[soc-family].inc` specifies the Linux kernel recipe by setting `PREFERRED_PROVIDER_virtual/kernel = linux-XXXX`.

See [imx-base.inc](#soc-family-inc-imxbase.inc) for an example of the `[soc-family].inc` recipe.

# <a name="section-6-0"></a> 6.0 `u-boot*`

This section describes `u-boot*.bb` and `u-boot*.bbappend` entities in the UML diagram [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0).
The discussion is applicable to all targets.

## <a name="section-6-1"></a> 6.1 `u-boot*.bb`: The top level `virtual/bootloader` control recipe

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) shows the `meta-[soc-vendor]` `u-boot*.bb` recipe used to build the bootloader. As discussed in [Section 5.2](../develop-mbl/5-0-machine-configuration-files.html#5-2-machine-conf-the-community-bsp-control-file), the `[soc-family].inc` defines `PREFERRED_PROVIDER_virtual/bootloader = u-boot-XXXX` to specify the boot loader recipe. The nominated boot loader recipe `u-boot-XXXX` (typically present in the `meta-[soc-vendor]` BSP layer) expresses its capability of being a `virtual/bootloader` provider by including `PROVIDES=virtual/bootloader` in the recipe. This relationship is expressed in [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) by the dotted-line arrow between `[soc-family].inc` and the interface symbol attached to `u-boot*.bb`.

## <a name="section-6-2"></a> 6.2 `u-boot*.bbappend` customization recipe

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) shows the `meta-[soc-vendor]-mbl` `u-boot*.bbappend` recipe used to
customize the `meta-[soc-vendor]` BSP layer `u-boot*.bb` recipe as required for MBL. Customization typically involves:
- Setting `SRC_URI` and `SRCREV` to point to a forked and patched version of U-Boot used for the target.
- Applying additional patches stored in `meta-[soc-vendor]-mbl`.
- Specifying new values of symbols to customize base recipe behavior.
- Handling device trees.

# <a name="section-7-0"></a> 7.0 `linux*`

This section describes the `linux*.bb` and `linux*.bbappend` entities in the UML diagram [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0).
The discussion is applicable to all targets.

## <a name="section-7-1"></a> 7.1 `linux*.bb`: The top level `virtual/kernel` control recipe

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) shows the `meta-[soc-vendor]` `linux*.bb` base recipe used to build the Linux kernel.

As discussed in [Section 5.2](../develop-mbl/5-0-machine-configuration-files.html#5-2-machine-conf-the-community-bsp-control-file), the `[soc-family].inc` defines `PREFERRED_PROVIDER_virtual/kernel = linux-XXX` to specify the kernel recipe.

The nominated Linux recipe `linux-XXXX` (typically present in the `meta-[soc-vendor]` BSP layer) expresses its capability of being a `virtual/kernel` provider by including `PROVIDES=virtual/kernel`
in the recipe. This relationship is expressed in [Figure 4.0](#figure-4-0) by the dotted-line arrow between `[soc-family].inc` and the interface symbol attached to `linux*.bb`.

## <a name="section-7-2"></a> 7.2 `linux*.bbappend` customization recipe

[Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) shows the `meta-[soc-vendor]-mbl` `linux*.bbappend` recipe used to customize the `meta-[soc-vendor]` BSP layer `linux*.bb` recipe as required for MBL. Customization typically includes:
- Setting `SRC_URI` and `SRCREV` to point to a forked and patched version of the Linux kernel with the required driver support and fixes.
- Applying additional patches stored in `meta-[soc-vendor]-mbl`.
- Specifying the default kernel configuration file to use using the `KBUILD_DEFCONFIG_<machine>` directive, for example, `KBUILD_DEFCONFIG_imx7s-warp-mbl ?= "warp7_mbl_defconfig"`.
- Merging kernel configuration fragments into the Linux configuration file to enable MBL-required kernel configuration, for example, to enable verified boot.
- Setting `INITRAMFS_IMAGE = "mbl-image-initramfs"`, to define the `meta-mbl` recipe for building `initramfs`.
- Setting `KERNEL_EXTRA_ARGS` to specify extra arguments supplied to the kernel.
- Setting other symbol values to customize base recipe behavior, for example, to report the current version of the kernel used by the target.

## <a name="section-7-3"></a> 7.3 `kernel.bbclass` `openembedded-core` support

This section provides detailed discussion of the `openembedded-core` meta-layer that provides support classes and recipes used by `linux*.bb` and `linux*.bbappend`.

<a name="figure-7-3"></a>


![figure-7-3](assets/mbl_oecore_kernel_classes_uml.png "Figure 7.3")
**Figure 7.3: The figure shows the `openembedded-core` `kernel.bbclass` hierarchy, including `mbl-fitimage`.**

[Figure 7.3](#figure-7-3) shows the UML diagram for the `kernel.bbclass` used to generate the Linux kernel, and how it relates to `linux*.bb(append)` and `mbl-fitimage.bbclass`. This is a more detailed representation of the `linux*` hierarchy shown in [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0), drawn to include more of the underlying `openembedded-core` support for building the kernel.

- **`linux*`**. This entity represents the `meta-[soc-vendor]` provided recipe for building the kernel. The recipe contains the line `inherit kernel` to inherit the `kernel.bblass` functionality.
- **`kernel`**.The `kernel.bbclass` implements the creation of the Linux kernel image (uImage by default).
  As can be seen from the diagram, the class hierarchy is not well composed because `kernel.bbclass` inherits from image specific base classes (such as `kernel-uimage.bbclass`), rather than image specific classes being specialized from a general purpose base class. However, this is a recognized problem and is a result of having to maintain backwards compatibility with an existing code base of working recipes. The general principal is that the infrastructure for generating kernel images has been partitioned into several logical parts coordinated through `kernel.bbclass`.
- **`linux-kernel-base`**. The `linux-kernel-base.class` provides helper functions to `kernel.bbclass` including extracting the Linux kernel version from `linux/version.h`.
- **`kernel-uimage`**. `KERNEL_CLASSES` defaults to `kernel-uimage` if unspecified, resulting in `kernel.bbclass` generating a uImage binary.
- **`kernel-arch`**. `kernel.bbclass` inherits from `kernel-arch.bbclass` to set the `ARCH` environment variable from `TARGET_ARCH`for building the Linux kernel.
- **`kernel-devicetree`**. `kernel.bbclass` inherits from `kernel-devicetree.bbclass` to generate the kernel device tree, deploying it to `DEPLOY_DIR_IMAGE`
- **`mbl-fitimage`**.`kernel.bbclass` is made to inherit from `mbl-fitimage.bbclass` by setting `KERNEL_CLASSES="mbl-fitimage"` in `${MACHINE}.conf` (see [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0), [Section 3.7](../develop-mbl/3-0-overview-of-mbl-yocto-meta-layers.html#3-7-yocto-bsp-recipe-software-architecture), and the next section for more details). Therefore, MBL does not use `kernel-uimage.bbclass`.
- **`kernel-fitimage`**. This is the base class for `mbl-fitimage.bbclass`, and is responsible for generating the FIT image according to configuration symbol settings.

## <a name="section-7-4"></a> 7.4 `kernel-fitimage.bbclass` and `mbl-fitimage.bbclass`

<a name="figure-7-4"></a>

![figure-7-4](assets/mbl_fit_image.png "Figure 7.4")

**Figure 7.4: The figure shows the `mbl-fitimage.bbclass` class hierarchy.**

[Figure 7.4](#figure-7-4) shows a UML class diagram annotated with the processing methods used in generating FIT images.
- **`kernel-fitimage.bbclass`**. The `kernel-fitimage.bbclass` encapsulates the `uboot-mkimage` tool invocation to combine a number of image components
  (such as kernel binary and DTB) into a single multi-component image (the FIT image). The class member functions `fitimage_emit_section_xxx()`
  write FIT image specification metadata sections in the fit-image description file (`fit-image.its`).
  The `fit_image_assemble()` member function is then used to generate the FIT image according to the `fit-image.its` specification.
  If `UBOOT_SIGN_ENABLE` is set (as is the case in MBL `${MACHINE}.conf` files), the `assemble` function signs the newly generated image (again using `uboot-mkimage`).
  Processing is hooked into the build by the class promoting certain member functions to task entry points.

- **`kernel-uboot.bbclass`**. This class is used to post-process the kernel image using the `objcopy` tool.
- **`uboot-sign.bbclass`**. This class is not used for signing because `mbl-fitimage.bbclass` processing is used instead.
- **`mbl-fitimage.bbclass`**. The `mbl-fitimage.bbclass` inherits from `kernel-fitimage.bbclass` and (re-)implements functions to customize
  the behaviour of the base class. See later in this section for more details.
- **`mbl-artefact-names.bbclass`**. This is a utility class used to define standard names for artifacts, for example, `MBL_UBOOT_CMD_FILENAME = "boot.cmd"`
  defines the U-Boot boot script file to be `boot.cmd` by default.


The main `kernel-fitimage.bbclass` member functions are:
- `__anonymous()`. This is an initialization function for the class that executes after parsing (the class constructor).
- `fitimage_emit_section_setup()`. Helper function to write the setup section in the FIT image `fit-image.its` file.
- `fitimage_emit_section_ramdisk()`. Helper function to write the `initramfs` section in the FIT image `fit-image.its` file.
- `fitimage_emit_section_config()`. Helper function to write the config section in the FIT image `fit-image.its` file.
- `fitimage_emit_section_dtb()`. Helper function to write the device tree binary section in the FIT image `fit-image.its` file.
- `fitimage_emit_section_kernel()`. Helper function to write the kernel section in the FIT image `fit-image.its` file.
- `fitimage_emit_section_maint()`. Helper function to write the main section in the FIT image `fit-image.its` file.
- `fitimage_assemble()`. Orchestrates the n-step procedure for writing the `fit-image.its` file by, depending on configuration, invoking the appropriate `fitimage_emit_section_xxx()` helper functions, creating the FIT image, and then signing the image.
- `do_assemble_fitimage()`. The class promotes this function to be a task entry point for the build process to create a FIT image, without `initramfs`.
- `do_assemble_fitimage_initramfs()`. The class promotes this function to be a task entry point for the build process to create a FIT image, including `initramfs`.

The key `${MACHINE}.conf` symbols controlling FIT image creation are as follows:

- `KERNEL_CLASSES`. Setting this symbol to `"mbl-fitimage"` results in the inclusion of `mbl-fitimage.bbclass` in the `kernel.bbclass` hierarchy as
  shown in [Figure 7.3](#figure-7-3). The processing is then hooked into the build.
- `UBOOT_SIGN_ENABLE`. Setting this symbol adds signing headers to the FIT image, according to MBL requirements.

The `mbl-fitimage.bbclass` member functions of interest are described briefly below:
- `fitimage_emit_section_boot_script()`. Helper function to write the boot script `fit-image.its` section, which incorporates the U-Boot `boot.cmd` file into the FIT image as the `boot.scr`.
- `fitimage_emit_section_config()`. This writes a modified form of the config to include the new `boot.scr` boot script section.
- `fitimage_assemble()`. This is a modified version of `kernel-fitimage.bbclass::fitimage_assemble()` to invoke the
`fitimage_emit_section_boot_script()` and `fitimage_emit_section_boot_config()` functions to add the `boot.scr`
and boot configuration to the FIT image.

# <a name="section-8-0"></a> 8.0 `atf-${MACHINE}.bb`

This section describes `the atf-${MACHINE}.bb`, `atf.inc` and `optee-os.bb` entities in the UML diagram [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0).
The discussion is applicable to all targets.

## <a name="section-8-1"></a> 8.1 `meta-[soc-vendor]-mbl` ATF recipes

In [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) the `meta-[soc-vendor]-mbl` machine configuration file `${MACHINE}.conf` orchestrates ATF creation by
specifying `PREFERRED_PROVIDER_virtual/atf = "atf-${MACHINE}"`. `atf-${MACHINE}.bb` includes `atf.inc` to create dependencies
on U-Boot and the kernel recipes.

ATF is dependent on U-Boot and the Linux kernel because:
- ATF packages U-Boot into the FIP image with other ATF build artifacts.
- ATF packages the U-Boot device tree including the FIT verification key into the FIP image.
- ATF may need to co-ordinate the location of shared memory buffers used for
  OP-TEE-Linux kernel inter-communication using overlays. ATF packages OP-TEE in the FIP image, whereas the kernel is packaged
  into the FIT image by `mbl-fitimage`.

The `atf.inc` dependency on the `virtual/bootloader` and `virtual/kernel` providers is created with a line in `atf.inc`:

    do_compile[depends] += " virtual/kernel:do_deploy virtual/bootloader:do_deploy optee-os:do_deploy"

This means that the `virtual/bootloader` and `virtual/kernel` artifacts should be deployed
before the `atf.inc do_compile()` method runs, so they are available for the ATF recipe to use.

<span class="notes">**Note:** `atf.inc` expects the `virtual/bootloader`, `virtual/kernel` and `optee*` artifacts on which it depends to be deployed to the `DEPLOY_DIR_IMAGE-${DEPLOY_DIR}/images/${MACHINE}/` directory. For the `imx7s-warp-mbl` target this directory is: `<workspace_root>/build-mbl/tmp-mbl-glibc/deploy/images/imx7s-warp-mbl`</span>

If required, ATF generates a ROT key pair used for signing artifacts. The ROT private key is also stored in the above directory. For more details about ATF root of trust key generation
and signing, see the [Mbed Linux OS Basic Signing Flow][basic-signing-flow].

## <a name="section-8-2"></a> 8.2 Details of the `meta-[soc-vendor]-mbl` ATF `atf-${MACHINE}.bb` recipe

<a name="Table-8-2"></a>

| Platform Name | ATF platform guide |
| --- | --- |
| NXP Warp7 | [warp7.rst][atf-doc-plat-warp7-rst] |
| NXP IMX8 Mini | [imx8.rst][atf-doc-plat-imx8-rst] |
| Raspberry Pi 3 | [rpi3.rst][atf-doc-plat-rpi3-rst] |
| Technexion Pico Pi | Not available |

**Table 8.2: The table shows the available ATF target documents available**.

The main source of information for assisting in the porting ATF to a new platform is the [ATF porting guide][atf-doc-plat-porting-guide]
and the associated ATF documents available in the repo. [Table 8.2](#Table-8-2) shows the ATF repository platform documents available, for example.

As an example, the ATF imx platform support is available here in the repository:

    ./drivers/imx
    ./plat/imx
    ./plat/imx/imx7
    ./plat/imx/imx8m
    ./plat/imx/common
    ./plat/imx/imx8qx
    ./plat/imx/imx8qm

One file of particular interest is `plat/imx/imx7/warp7/warp7_io_storage.c`, which defines the `plat_io_policy` descriptor for `imx7s-warp-mbl`:
```
    static const struct plat_io_policy policies[] = {
    #ifndef WARP7_FIP_MMAP
        [FIP_IMAGE_ID] = {
            &mmc_dev_handle,
            (uintptr_t)&mmc_fip_spec,
            open_mmc
        },
    #else
        [FIP_IMAGE_ID] = {
            &memmap_dev_handle,
            (uintptr_t)&fip_block_spec,
            open_memmap
        },
    #endif
        [BL32_IMAGE_ID] = {
            &fip_dev_handle,
            (uintptr_t)&bl32_uuid_spec,
            open_fip
        },
        [BL32_EXTRA1_IMAGE_ID] = {
            &fip_dev_handle,
            (uintptr_t)&bl32_extra1_uuid_spec,
            open_fip
        },
        [BL32_EXTRA2_IMAGE_ID] = {
            &fip_dev_handle,
            (uintptr_t)&bl32_extra2_uuid_spec,
            open_fip
        },
        [BL33_IMAGE_ID] = {
            &fip_dev_handle,
            (uintptr_t)&bl33_uuid_spec,
            open_fip
        },
    #if TRUSTED_BOARD_BOOT
        [TRUSTED_BOOT_FW_CERT_ID] = {
            &fip_dev_handle,
            (uintptr_t)&tb_fw_cert_uuid_spec,
            open_fip
        },
        [TRUSTED_KEY_CERT_ID] = {
            &fip_dev_handle,
            (uintptr_t)&trusted_key_cert_uuid_spec,
            open_fip
        },
        [TRUSTED_OS_FW_KEY_CERT_ID] = {
            &fip_dev_handle,
            (uintptr_t)&tos_fw_key_cert_uuid_spec,
            open_fip
        },
        [NON_TRUSTED_FW_KEY_CERT_ID] = {
            &fip_dev_handle,
            (uintptr_t)&nt_fw_key_cert_uuid_spec,
            open_fip
        },
        [TRUSTED_OS_FW_CONTENT_CERT_ID] = {
            &fip_dev_handle,
            (uintptr_t)&tos_fw_cert_uuid_spec,
            open_fip
        },
        [NON_TRUSTED_FW_CONTENT_CERT_ID] = {
            &fip_dev_handle,
            (uintptr_t)&nt_fw_cert_uuid_spec,
            open_fip
        },
    #endif /* TRUSTED_BOARD_BOOT */
    };
```
This is the starting point for porting ATF to a new platform.

# <a name="section-9-0"></a> 9.0 Example: `imx7s-warp-mbl` BSP recipe/package relationships

## <a name="section-9-1"></a> 9.1 Example: `imx7s-warp-mbl` recipe/package UML diagram

This section provides a concrete example of the UML diagram shown in [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) for the i.MX7 Warp7 target `MACHINE=imx7s-warp-mbl`.

<a name="figure-9-1"></a>

![figure-9-1](assets/mbl_warp7_uml_details.png "Figure 9.1")

**Figure 9.1: The UML diagram shows the relationships between the recipes and configuration files for the `imx7s-warp-mbl` target.**

[Figure 9.1](#figure-9-1) shows the `imx7s-warp-mbl` realization of recipes and configuration files shown in [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0).

This section will discuss the `meta-freescale` and `meta-freescale-3rdparty` entities shown in green in the above figure:
- **`imx7s-warp-mbl.conf`**. This is `meta-[soc-vendor]-mbl=meta-freescale-3rdparty-mbl` machine configuration file for the target.
    - `KERNEL_CLASSES  = "mbl-fitimage"`. The `mbl-fitimage.bbclass` is inherited into `kernel.bbclass` processing by defining this symbol to include `mbl-fitimage`.
    - `KERNEL_IMAGETYPE = "fitImage"`. The kernel is packages in a FIT image by specifying `"fitImage"`
    - `# KERNEL_DEVICETREE="imx7s-warp.dtb"`.  It's unnecessary to change this symbol here as the required `"imx7s-warp.dtb"` value is specified in `imx7s-warp.conf`.
    - `UBOOT_CONFIG = ""`
    - `UBOOT_MACHINE = "warp7_bl33_defconfig"`. This is the U-Boot default configuration file to use.
    - `UBOOT_CONFIG[sd] = ""`
    - `UBOOT_SUFFIX = "bin"`. This is used to enable U-Boot verified boot. See `uboot-sign.bbclass` for more information.
    - `UBOOT_BINARY = "u-boot.${UBOOT_SUFFIX}"`. This is the U-Boot binary name.
    - `UBOOT_ENTRYPOINT = "0x80800000"`. This is the U-Boot binary entry point.
    - `UBOOT_DTB_LOADADDRESS = "0x83000000"`. This is the location where the U-Boot DTD is loaded into memory.
    - `UBOOT_IMAGE = "mbl-u-boot.bin"` This is the name of the U-Boot image.
    - `UBOOT_SIGN_ENABLE = "1"`. This enables verified boot signing.
- **`imx7s-warp.conf`**. This is the `meta-[soc-vendor]=meta-freescale-3rdparty` machine configuration file that provides the base BSP support for the NXP Warp7 target.
- **`imx-base.inc`<a name="soc-family-inc-imxbase.inc"></a>**. This is an example of the `[soc-family].inc` file and gives the virtual provider definitions:
    - `PREFERRED_PROVIDER_virtual/bootloader="u-boot-fslc"`.
    - `PREFERRED_PROVIDER_virtual/kernel="linux-fslc"`.
- **`linux-fslc_${PV}.bb`**. This is the Freescale NXP community maintained mainline Linux kernel BSP recipe with backported features and fixes.
  The package version symbol `${PV}` is periodically updated to the next Linux kernel stable release version, for example, 4.9, 4.14, 4.19.
- **`linux-fslc.inc`**. This is a common include file for `linux-fslc*` recipes that specifies a Linux kernel default config, common dependencies
  and the inclusion of the `imx-base.inc` include file.
- **`linux-imx.inc`**. This is the common include file for IMX SoCs that encapsulates the interface to the `openembedded-core .bbclasses`, including
  `kernel.bbclass`.
- **`u-boot-fslc_${PV}.bb`**. This is the Freescale NXP community maintained mainline U-Boot BSP recipe with backported features and fixes.
  The package version symbol `${PV}` is periodically updated to the next U-Boot stable release version, for example, 2018.07, 2018.11.


## <a name="section-9.2"></a> 9.2 `imx7s-warp-mbl` recipe dependency graph

This section presents a recipe and machine configuration file dependency graph for the `imx7s-warp-mbl`
target as an alternative way of visualizing the information shown in [Figure 9.1](#figure-9-1).

<a name="figure-9-2"></a>
```
MACHINEOVERRIDES="armv7ve:imx:use-mainline-bsp:imx7s-warp:imx7s-warp-mbl:imx7s-warp-mbl"
MACHINE=imx7s-warp-mbl

  imx7s-warp-mbl.conf                                                                                 (1)
      KERNEL_XXX config                                                                               (2)
      KERNEL_CLASSES  = "mbl-fitimage"                                                                (3)
      KERNEL_IMAGETYPE = "fitImage"
      UBOOT_XXX config                                                                                (4)
      WKS_XXX config                                                                                  (5)
      IMAGE_BOOT_FILES config
      PREFERRED_PROVIDER_virtual/atf = "atf-${MACHINE}"                                               (6)
      |   \-> atf-imx7s-warp-mbl.bb
      |           DEPENDS = ""
      |           \-> aft.inc
      |               DEPENDS += " openssl-native coreutils-native optee-os u-boot virtual/kernel"
      |
      |
      \-> imx7s-warp.conf                                                                             (7)
          MACHINEOVERRIDES =. "mx7:mx7d:use-mainline-bsp:"                                            (8)
          KERNEL_DEVICETREE = "imx7s-warp.dtb"
          |
          \-> imx-base.inc                                                                            (9)
                # boot loader  recipe config
                PREFERRED_PROVIDER_u-boot ??= "u-boot-fslc"                                           (10)
                PREFERRED_PROVIDER_virtual/bootloader ??= "u-boot-fslc"                               (11)
                    \-> u-boot-fslc_XXXX.YY.bb                                                        (12)
                        \-> u-boot-fslc_%.bbappend                                                    (13)
                        \-> u-boot-fslc_%.bbappend                                                    (14)

                # kernel recipe config
                IMX_DEFAULT_KERNEL_use-mainline-bsp = "linux-fslc"                                    (15)
                PREFERRED_PROVIDER_virtual/kernel ??= "${IMX_DEFAULT_KERNEL}"
                    \-> linux-fslc_X.YY.bb                                                            (16)
                    |   \-> linux-fslc.inc                                                            (17)
                    |       \-> linux-imx.inc                                                         (18)
                    |       |       inherit kernel <others removed to save space>
                    |       |           \-> kernel.bbclass
                    |       |                   inherit ${KERNEL_CLASSES}                             (19)
                    |       |                       \-> mbl-fitimage.bbclass                          (20)
                    |       |                               inherit kernel-fitimage
                    |       |                                   \-> kernel-fitimage.bbclass
                    |       |
                    |       |                               do_compile[depends] += "mbl-boot-scr:do_deploy"
                    |       |                                                                         (21)
                    |       |
                    |       \-> u-boot-sign.bbclass                                                   (22)
                    |
                    \-> linux-fslc_%.bbappend                                                         (23)
                    \-> linux-fslc_%.bbappend                                                         (24)
```

**Figure 9.2: The diagram show the recipes and configuration files dependency graph for the `imx7s-warp-mbl`.**

[Figure 9.2](#figure-9-2) shows the recipes and machine configuration file dependency graph for the `imx7s-warp-mbl`:
- **(1)**. `meta-freescale-3rdparty-mbl/conf/machine/imx7s-warp-mbl.conf` is the `${MACHINE}.conf` configuration file for `imx7s-warp`.
  See [Figure 4.0](../develop-mbl/4-0-bsp-recipe-relationships.html#figure-4-0) and [Section 5.1](../develop-mbl/5-0-machine-configuration-files.html#5-1-machine-conf-the-top-level-bsp-control-file).
- **(2)**. The KERNEL_XXX symbols control Linux kernel and for FIT image generation.
  See [Section 7.4](../develop-mbl/7-0-linux.html#7-4-kernel-fitimage-bbclass-and-mbl-fitimage-bbclass) for more information.
- **(3)**. See (19).
- **(4)**. The UBOOT_XXX symbols control U-Boot image generation, and the signing of FIT image
    components by the uboot-mkimage tool.
- **(5)**. This specifies the WIC WKS kickstart file for the flash partition geometry.
- **(6)**. This specifies `atf-imx7s-warp-mbl.bb` is to be used as the `virtual/atf` provider.
- **(7)**. `meta-freescale-3rdparty/conf/machine/imx7s-warp.conf` is the `meta-[soc-vendor] ${machine}.conf` configuration file for `imx7s-warp-mbl`.
- **(8)**. `use-mainline-bsp` is used to configure `linux-fslc*`. See (15).
- **(9)**. `require meta-freescale/conf/machine/include/imx-base.inc`
- **(10)**. This makes our atf recipe work because we have DEPENDS += " u-boot "
- **(11)**. This specifies the `uboot-fslc` recipe to be the `virtual/bootloader` provider.
- **(12)**. `meta-freescale/recipes-bsp/u-boot/u-boot-fslc_2018.09.bb`, for example.
- **(13)**. `meta-freescale-3rdparty/recipes-bsp/u-boot/u-boot-fslc_%.bbappend`.
- **(14)**. `meta-freescale-mbl/recipes-bsp/u-boot/u-boot-fslc_%.bbappend`.
- **(15)**. Configured by MACHINEOVERRIDES including "use-mainline-bsp".
- **(16)**. `meta-freescale/recipes-kernel/linux/linux-fslc_4.18.bb`.
- **(17)**. `meta-freescale/recipes-kernel/linux/linux-fslc.inc`.
- **(18)**. `meta-freescale/recipes-kernel/linux/linux-imx.inc`.
- **(19)**. As kernel.bbclass includes the line:
        inherit ${KERNEL_CLASSES}
    and imx7s-warp-mbl.conf includes the line:
        KERNEL_CLASSES  = "mbl-fitimage"
    then the mbl-fitimage.bbclass is inherited by the `kernel.bbclass`.
- **(20)**. This generates the FIT image according to the MBL specification.
- **(21)**. This is how the dependency on `mbl-boot-scr` is introduced for the BSPs.
- **(22)**. This is used for FIT image signing.
- **(23)**. `meta-freescale-3rdparty/recipes-kernel/linux/linux-fslc_%.bbappend`.
- **(24)**. `meta-freescale-3rdparty-mbl/recipes-kernel/linux/linux-fslc_%.bbappend`.


# <a name="section-10-0"></a> 10.0 Summary of BSP porting tasks

This section provides a summary of the tasks required to integrate a pre-existing BSP for the new target `new-target` into MBL.

- Add the pre-existing `meta-[soc-vendor]` layer to `bblayers.conf` if required:
    - This layer should contain the `${machine}.conf` file called `new-target.conf` for the new target.
- Create the `u-boot*.bbappend` file:
    - Resolve licensing issues.
    - Upstream the U-Boot `new-target` port to `git://git.linaro.org/landing-teams/working/mbl/u-boot.git`.
    - Set `SRCREV` and `SRC_URI` for ported U-Boot.
    - Apply patches.
    - Fix DTB issues.
    - Upstream the `u-boot*.bbappend` recipe and associated files to `https://github.com/ARMmbed/meta-mbl`.
- Create the `linux*.bbappend` file:
    - Resolve licensing issues.
    - Upstream the Linux kernel `new-target` port to `git://git.linaro.org/landing-teams/working/mbl/linux.git`.
    - Set `SRCREV` and `SRC_URI` for ported Linux kernel.
    - Define default kernel configuration.
    - Merge required config to build with all required options.
    - Set `INITRAMFS_IMAGE = "mbl-image-initramfs"`.
- Manage Linux firmware files:
    - Resolve licensing issues.
    - Upstream the linux-firmware binary files to `git://git.linaro.org/landing-teams/working/mbl/linux-firmware.git`.
    - Modify `meta-mbl/openembedded-core-mbl/meta/recipes-kernel/linux-firmware/linux-firmware_%.bbappend` as required.
    - Upstream modified `linux-firmware_%.bbappend` recipe to `https://github.com/ARMmbed/meta-mbl`.
- Create the `optee-os.bbappend` recipe for building OP-TEE for the new target:
    - Resolve licensing issues.
    - Upstream the OP-TEE `new-target` port to `git://git.linaro.org/landing-teams/working/mbl/optee_os.git`.
    - Upstream the `optee-os.bbappend` recipe and associated files to `https://github.com/ARMmbed/meta-mbl`.
- Create the `atf-new-target-mbl.bb` recipe for building ATF for the new target:
    - Resolve licensing issues.
    - Upstream the ATF `new-target` port to `git://git.linaro.org/landing-teams/working/mbl/arm-trusted-firmware.git` or to
      `https://github.com/ARM-software/arm-trusted-firmware`.
    - Upstream modified `atf-new-target-mbl.bb` recipe to `https://github.com/ARMmbed/meta-mbl`.
- Create the ${MACHINE}.conf` file called `new-target-mbl.conf`:
    - Resolve licensing issues.
    - Define `PREFERRED_PROVIDER_virtual/atf = "atf-${MACHINE}`
    - Define `KERNEL_CLASSES  = "mbl-fitimage"`
    - Define `KERNEL_IMAGETYPE = "fitImage"`
    - Define `KERNEL_DEVICETREE = "XXX"` as required.
    - Define `UBOOT_ENTRYPOINT = "0xabcdefab"` as required.
    - Define `UBOOT_DTB_LOADADDRESS = "0xabcdefab"` as required.
    - Define `UBOOT_SIGN_ENABLE = "1"`
    - Define `WKS_FILE=${MACHINE}.wks`.
    - Upstream the `new-target-mbl.conf` machine configuration file to `https://github.com/ARMmbed/meta-mbl`.
    - Upstream the `new-target-mbl.wks` to `https://github.com/ARMmbed/meta-mbl`.

# <a name="section-11-0"></a> 11.0 References

* [ARM Trusted Firmware Platform Porting Guide][atf-doc-plat-porting-guide].
* [Mbed Linux OS Basic Signing Flow][basic-signing-flow].
* [OP-TEE documentation][optee-docs]
* [Embedded Linux Systems with the Yocto Project (Pearson Open Source Software Development Series) 1st Edition, Rudolf J. Streif,  ISBN-13: 978-0133443240 ISBN-10: 0133443248][strief-2016].
* [Linaro Connect 2016 Presentation LAS16-402 showing boot flow diagrams][linaro-connect-las16-402-slides].
* <a name="ref-tbbr-client"></a> Trusted Board Boot Requirements CLIENT (TBBR-CLIENT), Document number: ARM DEN0006C-1, Copyright ARM Limited 2011-2015.
* [U-Boot documentation][u-boot].
* [Yocto Project Board Support Package (BSP) Developer's Guide][yocto-project-board-support-package-bsp-developer-guide-latest]
* [Yocto Mega Manual][yocto-mega-manual-latest].


[android-verified-boot]:https://source.android.com/security/verifiedboot
[atf-doc-plat-warp7-rst]:https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/warp7.rst
[atf-doc-plat-imx8-rst]:https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/imx8.rst
[atf-doc-plat-rpi3-rst]:https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/rpi3.rst
[atf-doc-plat-porting-guide]:https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/getting_started/porting-guide.rst
[basic-signing-flow]:https://github.com/ARMmbed/meta-mbl/blob/mbl-os-0.7/docs/basic-signing-flow.md
[linaro-connect-las16-402-slides]:https://connect.linaro.org/resources/las16/las16-402/
[meta-linaro]:https://git.linaro.org/openembedded/meta-linaro.git/tree/
[meta-mbl]:https://github.com/ARMmbed/meta-mbl/blob/mbl-os-0.7
[meta-openembedded]:https://github.com/openembedded/meta-openembedded
[meta-raspberrypi]:http://git.yoctoproject.org/cgit/cgit.cgi/meta-raspberrypi/
[meta-virtualization]:http://git.yoctoproject.org/cgit/cgit.cgi/meta-virtualization/
[openembedded-core]:https://github.com/openembedded/openembedded-core
[optee-docs]:https://optee.readthedocs.io/
[strief-2016]:http://book.yoctoprojectbook.com/
[u-boot]:https://www.denx.de/wiki/view/DULG/UBootCmdGroupExec#Section_5.9.4.2.
[yocto-mega-manual-latest]:https://www.yoctoproject.org/docs/latest/mega-manual/mega-manual.html
[yocto-project-board-support-package-bsp-developer-guide-latest]:https://www.yoctoproject.org/docs/latest/bsp-guide/bsp-guide.html