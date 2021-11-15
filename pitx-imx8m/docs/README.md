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

# Errata

## Error: ethernet@30be0000 address not set

If the U-Boot does not has a valid MAC address set the network interface will
not work properly.

    U-Boot 2021.10-rc3-00004-g6cdd685132 (Sep 26 2021 - 07:20:52 +0200)
    CPU:   Freescale i.MX8MQ rev2.1 at 800 MHz
    Reset cause: POR
    Model: Kontron pITX-imx8m
    DRAM:  4 GiB
    MMC:   FSL_SDHC: 0, FSL_SDHC: 1
    Loading Environment from MMC... OK
    In:    serial
    Out:   serial
    Err:   serial
    Net:
    Error: ethernet@30be0000 address not set.

    Error: ethernet@30be0000 address not set.
    No ethernet found.


To find out the assigned mac address, the I2C EEPROM can be dumped:

    => i2c dev 0
    Setting bus to 0
    => i2c md 51 0 100
    0000: 00 33 50 10 09 74 6e 6f 6d 6f b5 2c ff ff ff ff    .3P..tnomo.,....
    0010: 0a 10 d0 00 18 a0 0d e2 00 4b 45 55 00 10 00 01    .........KEU....
    0020: 02 03 30 36 2f 30 33 2f 32 30 32 30 00 4d 4d 2f    ..06/03/2020.MM/
    0030: 44 44 2f 59 59 59 59 00 20 20 20 31 3a 41 43 00    DD/YYYY.   1:AC.
    0040: 00 00 d0 00 30 02 0f e0 00 01 02 03 04 05 00 00    ....0...........
    0050: ff ff ff 00 4b 6f 6e 74 72 6f 6e 20 45 75 72 6f    ....Kontron Euro
    0060: 70 65 20 47 6d 62 48 00 20 20 20 20 20 20 20 20    pe GmbH.
    0070: 20 20 70 49 54 58 2d 69 4d 58 38 4d 00 20 42 30      pITX-iMX8M. B0
    0080: 30 00 55 54 44 30 38 30 30 30 32 00 20 20 20 20    0.UTD080002.
    0090: 34 34 30 31 31 2d 30 34 30 38 2d 31 33 2d 34 00    44011-0408-13-4.
    00a0: 00 00 d0 00 11 a4 0b e2 04 4b 45 55 00 10 01 01    .........KEU....
    00b0: 30 30 3a 65 30 3a 34 62 3a 36 66 3a 64 63 3a 63    00:e0:4b:6f:dc:c
    00c0: 64 00 00 00 f2 00 03 67 c5 00 00 00 81 00 00 00    d......g........
    00d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
    00e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
    00f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
    =>

Here we have the `00:e0:4b:6f:dc:cd` value as macadress.
Set this value to the environment variable `ethaddr` and save the environment.

    => setenv ethaddr 00:e0:4b:6f:dc:cd
	=> saveenv


## Error: the RTC has no battery connected

When no battery is connected to the RTC the access to the RTC will lead to the
following Warning message and a no valid data will be returned.

    => date
    ### Warning: temperature compensation has stopped
    ### Warning: Voltage low, data is invalid
    ## Get date failed

To avoid this the RTC state can be reset:

    => date
    ### Warning: temperature compensation has stopped
    ### Warning: Voltage low, data is invalid
    ## Get date failed
    => date reset
    Reset RTC...
    Date: 2000-01-01 (Saturday)    Time:  0:00:00
    => date
    Date: 2000-01-01 (Saturday)    Time:  0:00:05

# Installing a distribution

## openSUSE-tumbleweed

To avoid an error during the installation disable the `Update NVRAM` boot option.

    YaST2 - installation @ install

      Installation Settings                                    [Release Notes...]
      Click a headline to make changes or use the "Change..." menu below.
      ┌──────────────────────────────────────────────────────────────────────────┐
      │Booting                                                                   ┬
      │                                                                          │
      │ *  Boot Loader Type: GRUB2 EFI                                           │
      │ *  Secure Boot: enabled (disable)                                        ┴
      │ *  Update NVRAM: disabled (enable)                                       │
      │                                                                          │
      │Software                                                                  │
      │                                                                          │
      │ *  Product: openSUSE Tumbleweed                                          │
      │ *  Patterns:                                                             │
      │     +  Help and Support Documentation                                    │
      │     +  Minimal Base System                                               │
      │     +  Enhanced Base System                                              │
      │     +  AppArmor                                                          │
      └──────────────────────────────────────────────────────────────────────────┘
                                      [Change...↓]
     [ Help  ]              [ Back  ]              [Abort]              [Install]

## Debian 11

Debian GRUB installation must be finalized manually. During Debian installation,
the "Install the GRUB boot loader" step will fail and manual intervention
is necessary to make the installed system bootable.

Select "Execute a shell" and do:

	# chroot /target
	# update-grub
	# mkdir /boot/efi/EFI/BOOT
	# cp -v /boot/efi/EFI/debian/grubaa64.efi /boot/efi/EFI/BOOT/bootaa64.efi

Exit the chroot and the shell to return to the installer and select the
"Continue without boot loader" step to finish installation.
