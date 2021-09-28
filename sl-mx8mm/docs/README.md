# Building and flashing the firmware

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

This will create a file named `flash.bin` which you have to copy to the SD card
at offset 33k.

You can use a SD card reader on a linux host:

	dd if=flash.bin of=/dev/sd[x] bs=1k seek=33
