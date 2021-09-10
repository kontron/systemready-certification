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
	CROSS_COMPILE=aarch64-linux-gnu- make -j4

This will create a file named `flash.bin` wich you have to copy to the SD card
at offset 33k.

You can use a SD card reader on a linux host system:

    dd if=flash.bin of=/dev/sd[x] bs=1k seek=33

