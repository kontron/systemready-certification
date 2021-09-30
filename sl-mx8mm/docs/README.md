# Building the firmware

	cd arm-truste-firmware
	CROSS_COMPILE=aarch64-linux-gnu- make PLAT=imx8mm IMX_BOOT_UART_BASE="0x30880000" bl31
	cp build/imx8mm/release/bl31.bin ../u-boot
	cd ..

	wget https://www.nxp.com/lgfiles/NMG/MAD/YOCTO/firmware-imx-8.9.bin
	chmod +x firmware-imx-8.9.bin
	./firmware-imx-8.9.bin
	cp firmware-imx-8.9/firmware/ddr/synopsys/lpddr4*.bin u-boot

	cd u-boot
	git am ../pending-patches/u-boot/*.patch
	CROSS_COMPILE=aarch64-linux-gnu- make kontron-sl-mx8mm_defconfig
	CROSS_COMPILE=aarch64-linux-gnu- make -j4

This will create a file named `flash.bin`, `spi-flash.bin`  and
`firmware-update.itb` that will be used in the next steps.

# Flashing the firmware

The sl-mx8mm boots either from SD card or from the SPI flash.

The decision from where to boot is automatically done:

1. try to boot from SD card
2. try to boot from SPI flash

For tests we have to make sure we start the board from SPI flash. So do not
insert a bootable SD card.

## Write to SD card

The `flash.bin` can be use to copy it to the SD card.
For the SD card you have to write the file to offset `33k`.

You can use a SD card reader on a linux host to write the file:

	dd if=flash.bin of=/dev/sd[x] bs=1k seek=33

## Write to SPI flash

The `spi-flash.bin` can be to update the SPI flash:

	=> setenv autoload no
    => dhcp
    => tftp $loadaddr <TFTPSERVERIP>:spi-flash.bin
	=> sf probe
	=> sf update $loadaddr 0x0 ${filesize}

# Create update capsule

During building the u-boot above also the `firmware-update.itb` will be created.
The update image will update the U-boot in the SPI flash. So make sure you boot
from SPI to verify that the update works.

This can be used as input to create the EFI capsule file. To verify that
the UpdateCapsule mechanism works, 2 different firmware binaries should
be created. To be able to distinguish them, different `BUILD_TAG`s are assigned.

    # BUILD_TAG=capsule1 CROSS_COMPILE=aarch64-linux-gnu- make -j4
    # ./tools/mkeficapsule --fit firmware-update.itb --index 1 capsule1.bin
    # BUILD_TAG=capsule2 CROSS_COMPILE=aarch64-linux-gnu- make -j4
    # ./tools/mkeficapsule --fit firmware-update.itb --index 1 capsule2.bin

## Using efidebug to directly update (Testing purpose)

Load the update file to memory and do the update

    => tftp $loadaddr <SERVERIP>:capsule1.bin
    => efidebug capsule update -v ${loadaddr}
