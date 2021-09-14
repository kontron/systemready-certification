# Building and flashing the firmware

    cd arm-trusted-firmware
    CROSS_COMPILE=aarch64-linux-gnu- make PLAT=ls1028ardb bl31
    cp build/ls1028ardb/release/bl31.bin ../u-boot
    cd ..

    cd u-boot
    git am ../pending-patches/u-boot/*.patch
    CROSS_COMPILE=aarch64-linux-gnu- make kontron_sl28_defconfig
    # enable loading of bl31
    echo CONFIG_SL28_SPL_LOADS_ATF_BL31=y >> .config
    CROSS_COMPILE=aarch64-linux-gnu- make olddefconfig
    CROSS_COMPILE=aarch64-linux-gnu- make -j4

This will create a file named `u-boot.rom` which you have to flash on the
SPI flash at offset `210000h`. Either by copying it onto an USB thumb
drive, an SD card or on a tftp server.

In the u-boot CLI do the following:

    load usb 0:1 $loadaddr u-boot.rom
    sf probe 0
    sf update $fileaddr 210000 $filesize
    reset
