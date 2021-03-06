# Development image build examples

<span class="tips">**Tip**: If you downloaded an evaluation image, you can skip the build stage [and go directly to writing](../first-image/writing-an-image-to-supported-boards.html).</span>

The following examples assume:

* You have an output directory for your machine, for example:
    * `./artifacts-pico7`
    * `./artifacts-nxpimx`
    * `./artifacts-pico6`
    * `./artifacts-rpi3`
* You have an [SSH agent](../first-image/development-environment.html) to access your own private repositories. For more information, see [the GitHub SSH documentation](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/).

The following are examples of build commands to produce development images for supported platforms.

## PICO-PI with i.MX7D

```
./mbl-tools/build/run-me.sh --builddir ./build-pico7 --outputdir ./artifacts-pico7 -- --machine imx7d-pico-mbl --branch mbl-os-0.10
```

## PICO-PI with i.MX6UL

```
./mbl-tools/build/run-me.sh --builddir ./build-pico6 --outputdir ./artifacts-pico6 -- --machine imx6ul-pico-mbl --branch mbl-os-0.10
```

## NXP i.MX8M Mini EVK

```
./mbl-tools/build/run-me.sh --builddir ./build-nxpimx --outputdir ./artifacts-nxpimx -- --machine imx8mmevk-mbl --branch mbl-os-0.10
```

## Raspberry Pi 3

```
./mbl-tools/build/run-me.sh --builddir ./build-rpi3 --outputdir ./artifacts-rpi3 -- --machine raspberrypi3-mbl --branch mbl-os-0.10
```


***

Copyright © 2020 Arm Limited (or its affiliates)
