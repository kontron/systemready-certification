# Building and flashing the firmware

	cd arm-trusted-firmware
	CROSS_COMPILE=aarch64-linux-gnu- make PLAT=imx8mq bl31
	cp build/imx8mq/release/bl31.bin ../u-boot/
	cd ..

	wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-imx-8.11.bin
	chmod +x firmware-imx-8.11.bin
	./firmware-imx-8.11.bin
	cp firmware-imx-8.11/firmware/ddr/synopsys/lpddr4*.bin u-boot
	cp firmware-imx-8.11/firmware/hdmi/cadence/signed_hdmi_imx8m.bin u-boot

	cd u-boot
	git am ../pending-patches/u-boot/*.patch
	CROSS_COMPILE=aarch64-linux-gnu- make kontron_pitx_imx8m_defconfig
	CROSS_COMPILE=aarch64-linux-gnu- make

This will create a file named `flash.bin` and `firmware-update.itb` that will
be used in the next steps.

The `flash.bin` can be use to copy it to the SD card or the eMMC at offset `33k`.

## Flashing and Booting from eMMC

The pitx-imx8m support booting either from SD card or from onboard eMMC.
For the tests we need to install the `flash.bin` to the eMMC boot partition.

To select the boot source you have to do the DIP switch (SW1 is located near
the MiniDP connector) setting as followed:

| SW1.1 | SW1.2 | Source  |
| ----- | ----- | ------- |
| OFF   | OFF   | SD card |
| OFF   | ON    | eMMC    |

Install the `flash.bin` to eMMC boot partion.

We assume that the `flash.bin` was copied copied into memory at `$loadaddr`.
For example by downloading the file via tftp from a server (a) or by copying
from an external storage like a partion on an USB flash drive (b).

Option a) Copy the the `flash.bin` file onto a tftp server that is reachable
from the pitx-imx8m network. Then load the file to `$loadaddr`.

    => setenv autoload no
	=> dhcp
	=> tftp $loadaddr <TFTPSERVERIP>:flash.bin

Option b) Copy the file to a partition on a USB flash drive. In the example
we used the first partion on the flash drive. Then insert the USB flash drive
into an USB port of the pitx-im8m and load the file to `$loadaddr`.

    => usb start
	=> load mmc 0:1 $loadaddr flash.bin

Now you can flash the content in `$loadaddr` onto the eMMC boot partiion.

    => mmc dev 0 1
    => mmc write $loadaddr 0x42 0x1000

Configure the eMMC to boot from the boot partion:

    => mmc partconf 0 0 1 0

# Create update capsule

During building the u-boot above also the `firmware-update.itb` will be created.

This can be used as input to create the EFI capsule file. To verify that
the UpdateCapsule mechanism works, 2 different firmware binaries should
be created. To be able to distinguish them, different `BUILD_TAG`s are assigned.

    # BUILD_TAG=capsule1 CROSS_COMPILE=aarch64-linux-gnu- make -j4
    # ./tools/mkeficapsule --fit firmware-update.itb --index 1 capsule1.bin
    # BUILD_TAG=capsule2 CROSS_COMPILE=aarch64-linux-gnu- make -j4
    # ./tools/mkeficapsule --fit firmware-update.itb --index 1 capsule2.bin


## Using efidebug to directly update (Testing purpose)

Load the update file from storage

	=> usb start
    starting USB...
    Bus usb@38100000: Register 2000140 NbrPorts 2
    Starting the controller
    USB XHCI 1.10
    Bus usb@38200000: Register 2000140 NbrPorts 2
    Starting the controller
    USB XHCI 1.10
    scanning bus usb@38100000 for devices... 1 USB Device(s) found
    scanning bus usb@38200000 for devices... 4 USB Device(s) found
           scanning usb for storage devices... 1 Storage Device(s) found
    => ls usb 0:1
                EFI/
                grub/
     32930304   Image
     92389888   ramdisk-busybox.img
          298   grub.cfg
      1 80688   firmware-update.itb
      1080780   capsule1.bin

    5 file(s), 2 dir(s)

    =>
	=> load usb 0:1 $loadaddr capsule1.bin
    1080780 bytes read in 18 ms (57.3 MiB/s)

Do the update

    => efidebug capsule update -v ${loadaddr}

This will update the firmware in the eMMC boot partion and after a reset the
new firmware version will start.
