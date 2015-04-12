# Globalsale Mirabox Unofficial Documentation

There's not a lot of good documentation about Globalscale Technology's [Mirabox][1] so this repo is intended to be a place to document and share what the community knows about it.

This was originally a gist (https://gist.github.com/psanford/6416725) but I've moved it to a full repository so that it is easier for others to contribute to.

## Hardware
The Mirabox uses a Marvell Armada 370 SoC ARM chip. The Armada 370 uses the ARMv7-A architecture[2]. The Mirabox comes with:[1]

- 1GB of RAM
- 1GB of NAND flash
- 2 10/100/1000 ethernet ports
- 2 USB 3.0 host ports
- 1 microsd card reader (2? 1 external and 1 internal)
- 1 Mini PCIe expansion slot (taken by Wifi chip on latest revision)
- 3 GPIO LEDs
- 1 button (older models this was a GPIO button, newer ones it is a hardware reset)
- 802.11b/g/n Wifi (88W8766 or 88W8787)
- Bluetooth 4.0 (88W8766 or 88W8787) (older hardware revs had BT 3.0)


In June 2014 Marvell released detail specs for the Armada 370 SoC:

- Product Brief:   http://www.marvell.com/embedded-processors/armada-300/assets/Marvell_ARMADA_370_SoC.pdf
- Hardware Spec:   http://www.marvell.com/embedded-processors/armada-300/assets/ARMADA370-datasheet.pdf
- Functional Spec: http://www.marvell.com/embedded-processors/armada-300/assets/ARMADA370-FunctionalSpec-datasheet.pdf

## Old vs new hardware

There have been at least 3 hardware revisions of the Mirabox hardware. The following is based on my observations of different hardware units and is probably incomplete:

### Rev1:

- Metal side with "Mirabox" machined into it
- Button on a GPIO line
- 88W8787 combo Wifi/Bluetooth


### Rev2

- Plastic side with adhesive "Mirabox - LX" label
- Button is hardware reset


### Rev3

- Plastic side with adhesive "Mirabox - LX" label
- Button is hardware reset
- 88W8766 Wifi/Bluetooth chip using the PCIe slot (the chip uses the USB bus)
- No internal sdcard slot (the fold type); the external slot is still there


## Kernel
The Mirabox originally came with a modified 2.6.35.9 kernel. More recent versions now ship with a modified 3.2.36 kernel.

Fortunately there is good support for the Armada 370 in more recent versions of the linux kernel.

What works in >= 3.14 kernel?

- USB
- LEDs
- Ethernet*
- SDCard
- Wireless**
- NAND flash (since 3.14)

*Ethernet works properly, but by default the NICs will have randomly assigned mac addresses. This can be done via uBoot[10][10] if the mvneta is compiled into the kernel (not as a module). You can also read the uBoot variables for ethaddr and eth1addr using fw_printenv and set the mac addresses from an init script.

**Wireless works, but for the 88W8787 chip you must load the mvsdio kernel module with nodma=1 option[11][11].

### Building a kernel
You will need a cross compiling toolchain to build the kernel. The easiest way to get that is by downloading the one provided by [Globalscale][3]. To actually build the kernel run the following commands:

    export PATH="/home/build/armv7-marvell-linux-gnueabi/bin:$PATH"
    make ARCH=arm CROSS_COMPILE=arm-marvell-linux-gnueabi- mvebu_v7_defconfig
    make ARCH=arm CROSS_COMPILE=arm-marvell-linux-gnueabi- zImage
    make ARCH=arm CROSS_COMPILE=arm-marvell-linux-gnueabi- armada-370-mirabox.dtb
    cp arch/arm/boot/zImage zImage-with-dtb
    cat arch/arm/boot/dts/armada-370-mirabox.dtb >> zImage-with-dtb
    ./scripts/mkuboot.sh -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n 'Linux-marvell' -d zImage-with-dtb uImage

The uBoot version that ships with the Mirabox doesn't support device tree, you you need to append the device tree to the kernel (and select the `Use appended device tree` option in the kernel config.

## Bootloader
The Mirabox comes with an old version of [U-Boot][4], modified to support the hardware. Currently there is no support in mainline U-Boot for the Mirabox.

Initial work has been done to support the Mirabox in the [Barebox Bootloader][5] by [Free Electrons][6]. This is currently limited to booting via serial port[7].

## Building Barebox
You will need to extract a binary blob from an existing bootloader for the mirabox. The easiest way to do this is to download the u-boot image from [Globalscale][3].

    # cd into barebox root dir
    mkdir /tmp/mirabox
    cd scripts
    make kwbimage
    ./kwbimage -x -i ~/downloads/u-boot-db88f6710bp_nand.bin -o /tmp/mirabox

To build barebox:

    # cd into barebox root dir
    export PATH="/home/build/armv7-marvell-linux-gnueabi/bin:$PATH"
    cp /tmp/mirabox/kwbimage.cfg kwbimage.cfg
    sed -i -e 's/nand/uart/' kwbimage.cfg
    make ARCH=arm CROSS_COMPILE=arm-marvell-linux-gnueabi- mvebu_defconfig
    make ARCH=arm CROSS_COMPILE=arm-marvell-linux-gnueabi-

To boot to barebox, run the following command:

    # cd into barebox root dir
    ./scripts/kwboot -t -b barebox-flash-image  -B 115200 /dev/ttyUSB0

### What are kwbimage and kwboot?
Marvell EBU SoC supports booting an image over UART[8], kwboot is a tool that implements this protocol.

The Marvell SoC requires a binary blob on boot for DDR3 training[9]. This is bundled with the bootloader in a specific format. kwbimage can be used to extract that binary blob from one image and add it to a new bootloader image.

## Resources
Source and binaries from Globalscale can be found at:

- http://www.plugcomputer.org/downloads/mirabox/
- https://code.google.com/p/mirabox/downloads/list

## Thanks
I want to thank the folks at [Free Electrons][6] for the work they have done to support the Armada 370 and the Mirabox.

[1]: http://www.globalscaletechnologies.com/p-58-mirabox-development-kit.aspx
[2]: http://wiki.openwrt.org/doc/hardware/soc/soc.marvell
[3]: http://www.plugcomputer.org/downloads/mirabox/
[4]: http://www.denx.de/wiki/U-Boot
[5]: http://barebox.org
[6]: http://free-electrons.com
[7]: http://free-electrons.com/blog/barebox-2013-07/
[8]: http://git.pengutronix.de/?p=barebox.git;a=commit;h=0535713bbfa059c1bc20da24d33bb183c4f555dc
[9]: http://git.pengutronix.de/?p=barebox.git;a=commit;h=6bb3a08cd3864f3ee1b9f4becf26b55ac9c0a524
[10]: http://lists.infradead.org/pipermail/linux-arm-kernel/2013-September/196688.html
[11]: http://lists.infradead.org/pipermail/linux-arm-kernel/2013-March/158753.html
