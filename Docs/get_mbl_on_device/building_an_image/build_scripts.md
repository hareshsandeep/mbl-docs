# Building an MBL developer image

<span class="notes">**Note**: Mbed Linux OS is currently in limited preview. If you would like access to the code repositories, [please request to join the preview](https://os.mbed.com/linux-os/).</span>

<span class="tips">**Tip**: If you downloaded an evaluation image, you can skip the build stage [and go directly to writing]().</span>

Please note that each release has its own branch. Throughout this guide, the release branch is assumed to be `mbl-os-0.6`.

## Building scripts

<span class="notes">**Note**: You need to use an SSH agent to build. For usage, see [the GitHub SSH documentation](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/).</span>


If you haven't already done so, check out the relevant branch from the `build-mbl` repository (in this example, we use `mbl-os-0.6`):

```
$ git clone git@github.com:ARMmbed/mbl-tools.git --branch mbl-os-0.6
```

The repository includes the `run-me.sh` script, which:

1. Creates and launches a Docker container that encapsulates the MBL build environment.
1. Launches a build script, `build.sh`, inside the container.

The `run-me.sh` and `build.sh` scripts are called in a single command, with options that control what to build and how. The general form of a `run-me.sh` invocation is:

```
./mbl-tools/build-mbl/run-me.sh [RUN-ME.SH OPTIONS]... -- [BUILD.SH OPTIONS]...
```

<span class="tips">Note the use of `--`. It separates options for `run-me.sh` from options for `build.sh`.</span>

To invoke the `run-me.sh` help menu, use:

```
./mbl-tools/build-mbl/run-me.sh -h
```

To invoke the `build.sh` help menu, use:

```
./mbl-tools/build-mbl/run-me.sh -- -h
```

The following build options are mandatory:

| Name | Information |
| --- | --- |
| `--branch` | Select the MBL branch to build. For example, to build the branch `mbl-os-0.6`: <br>`./mbl-tools/build-mbl/run-me.sh -- --branch mbl-os-0.6 --machine raspberrypi3-mbl` |
| `--machine` | Select the target device. <br>The options are [**PICO-PI with IMX7D**, `imx7d-pico-mbl`], [**NXP 8M Mini EVK**, `imx8mmevk-mbl`], [**Warp7**, `imx7s-warp-mbl`] and [**Raspberry Pi 3**, `raspberrypi3-mbl`]. <br>Example: `./mbl-tools/build-mbl/run-me.sh -- --machine <MACHINE>` |
| `--builddir` | Create a build directory. This option is for `run-me.sh`. <br>You must use a different build directory for every device (machine), and we recommend including the device's name in the directory's name. <br>Note that this directory includes all other artifacts, such as build and error logs. For example, if you've created `mkdir /path/to/my-build-dir`, the builddir will be `./mbl-tools/build-mbl/run-me.sh --builddir /path/to/my-build-dir` |
| `--outputdir` | Specify the output directory for all build artifacts (pinned manifest, target specific images etc). <br>For example, if you're created `mkdir /path/to/artifacts`, the outpudir will be `./mbl-tools/build-mbl/run-me.sh --outputdir /path/to/artifacts` |

An example using all mandatory options:

```
./mbl-tools/build-mbl/run-me.sh --builddir /path/to/builddir --outputdir /path/to/artifacts -- --branch mbl-os-0.6 --machine <MACHINE>
```

The following build options are not mandatory, but you may find that they improve the development process:

| Name | Information |
| --- | --- |
| `--downloaddir` | Cache downloaded artifacts between successive builds (do not use cacheing for parallel builds). <br>For example, if you've created `mkdir /path/to/downloads`, the downloaddir will be `./mbl-tools/build-mbl/run-me.sh --downloaddir /path/to/downloads` |
| `--external-manifest` | You can build using a pinned manifest, which is an encapsulation created by a build and containing enough information to allow an exact rebuild. The manifest is created in your output directory (`outputdir`). <br>To use it to rebuild, run `./mbl-tools/build-mbl/run-me.sh --external-manifest /path/to/pinned-manifest.xml` |