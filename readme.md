### Porting imx swupdate on imx8qm imx-5.15.71_2.2.1 yocto

　

#### SWUpdate ：

SWUpdate offical site : https://sbabic.github.io/swupdate/index.html


The NXP SWUpdate solution ：

- [AN13872 : Enabling SWUpdate on i.MX 6ULL](https://www.nxp.com/webapp/Download?colCode=AN13872&location=null)　⭐ This document is rich helpful to understand the basic of swupdate

- [SWUpdate OTA i.MX8MM EVK / i.MX8QXP MEK](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/SWUpdate-OTA-i-MX8MM-EVK-i-MX8QXP-MEK/ta-p/1307416)


　


#### Porting guide ： imx8qm imx-5.15.71_2.2.1 yocto

1. Clone imx-5.15.71_2.2.1 yocto project.

```
    // imx-5.15.71_2.2.1 yocto
    ${nxp}/imx_5.15.71_2.2.0-repo.tar.bz2
```

2. Clone meta-swupdate.

```
    cd ${imx-yocto-bsp}/source
    git clone https://github.com/sbabic/meta-swupdate.git -b kirkstone
```

3. Clone meta-swupdate-imx.

```
    cd ${imx-yocto-bsp}/source
    git clone https://github.com/nxp-imx-support/meta-swupdate-imx.git -b kirkstone_5.15.71_2.2.0

    modify u-boot-imx recipe ：

      #. cd ${imx-yocto-bsp}/sources/meta-swupdate-imx/recipes-bsp/u-boot/files

         rm 0001-enable-env_redunand-bootcount-limit-LF_v5.15.71-2.2.0.patch
         rm 0002-default-env-LF5.15.71-2.2.0.patch
         add the new patch, 0001-enable-env_redunand-bootcount-limit-default-env-LF5..patch

      #. edit ${imx-yocto-bsp}/sources/meta-swupdate-imx/recipes-bsp/u-boot/u-boot-imx_%.bbappend

           modify SRC_URI += " file://0001-enable-env_redunand-bootcount-limit-default-env-LF5..patch"

            +do_install:append:mx8qm-nxp-bsp () {
            +
            +    echo "/dev/mmcblk0 0x400000 0x2000" > ${D}/${sysconfdir}/fw_env.config
            +    echo "/dev/mmcblk0 0x402000 0x2000" >> ${D}/${sysconfdir}/fw_env.config
            +    echo "${MACHINE} ${SWU_HW_REV}" > ${D}/${sysconfdir}/hwrevision
            +}
            +
```

4. Modify Yocto source by adding layers for `meta-swupdate` and `meta-swupdate-imx`.
```
    edit ${imx-yocto-bsp}/sources/meta-imx/tools/imx-setup-release.sh
    +echo "BBLAYERS += \"\${BSPDIR}/sources/meta-swupdate\"" >> $BUILD_DIR/conf/bblayers.conf
    +echo "BBLAYERS += \"\${BSPDIR}/sources/meta-swupdate-imx\"" >> $BUILD_DIR/conf/bblayers.conf
```

5. Modify Yocto source by adding packages to the image installation list.
```
    edit ${imx-yocto-bsp}/sources/meta-imx/tools/imx-setup-release.sh
    +echo "IMAGE_INSTALL:append = \" lua swupdate swupdate-www swupdate-progress swupdate-client swupdate-tools-ipc u-boot-imx u-boot-fw-utils systemd-swusys json-c\"" >> conf/local.conf
    +echo "IMAGE_FSTYPES:append = \" ext4 ext4.gz wic.bmap wic.gz\"" >> conf/local.conf

```

6. Build SWUpdate inside images from Yocto.

```
    cd ${imx-yocto-bsp}/
    DISTRO=fsl-imx-wayland MACHINE=imx8qmmek source imx-setup-release.sh -b build_fsl-imx-wayland
    bitbake imx-image-full
    bitbake swupdate-image
```


</br>
</br>　


#### Build the base image, the update image by imx swupdate-scripts ：

1. Clone NXP swupdate-scripts

```
    git clone https://github.com/NXP/swupdate-scripts
```

2. Modify swupdate-scripts to adapt imx8qm

```
   cd ${swupdate-scripts}

   apply patch 0001-modify-swupdate-scripts-to-adapt-imx8qm-single-copy-.patch

   the patch is to do :

   #. new board name "imx8qm" into ${swupdate-scripts}/boards/cfg_boards.cfg

   #. new config/template files into ${swupdate-scripts}/boards for support imx8qm single copy, double copy

        cfg_imx8qm_base_singlecopy.cfg
        cfg_imx8qm_update_image_singlecopy.cfg
        sw-description-imx8qm-emmc-singlecopy-image.template

        cfg_imx8qm_base_dualcopy.cfg
        cfg_imx8qm_update_image_dualcopy.cfg
        sw-description-imx8qm-emmc-dualcopy-image.template

   #. modify scripts

        update_image_build/emmc_bootpart.sh
        update_image_build/env_set_bootslot.sh
        update_image_build/swu_update_image_build.sh
        utils/utils.sh

   #. modify symbolic link to point to the target image

       e.g. point to imx8qm target image

        base_image_assembling/slota -> ${imx-yocto-bsp}/build_cnh-pegasus_xwayland_imx-image-full/tmp/deploy/images/cnh-pegasus/
        base_image_assembling/slotb -> ${imx-yocto-bsp}/build_cnh-pegasus_xwayland_imx-image-full/tmp/deploy/images/cnh-pegasus/

        update_image_build/slot_update -> ${imx-yocto-bsp}/build_cnh-pegasus_xwayland_imx-image-full/tmp/deploy/images/cnh-pegasus/

   #. make symbolic link to point to the config file depends on the choose of single copy / double copy

        boards/cfg_ima8qm_base.cfg -> cfg_imx8qm_base_singlecopy.cfg or cfg_imx8qm_base_doublecopy.cfg
        boards/cfg_ima8qm_update_image.cfg -> cfg_ima8qm_update_image_singlecopy.cfg or cfg_ima8qm_update_image_doublecopy.cfg

```

3. Execute scripts to build base image , update image

```
    #. build base image by assemble_base_image.sh:

        assemble_base_image.sh - generate update image
        -o specify output image name. Default is swu_<SLOT>_rescue_<soc>_<storage>_<date>.sdcard
        -d enable double slot copy. Default is single slot copy.
        -e enable emmc. Default is sd.
        -b soc name. Currently, imx93, imx8mm and imx6ull are supported.
        -m Only regenerate or overwrite MBR. This option can be used to generate MBR individually.
        Suppose that we don't need to generate MBR every time. Normally we only need to generate it once.
        -h print this help.

        single copy :  assemble_base_image.sh -e -b imx8qm
        double copy :  assemble_base_image.sh -d -e -b imx8qm

        the output image is e.g. swu_singlecopy_rescue_imx8qm_emmc_20231226.sdcard  or
                                 swu_doublecopy_rescue_imx8qm_emmc_20231226.sdcard


    #. build update image by swu_update_image_build.sh

        first must use swu_generate_part_img.sh to prepare the boot partition used by swu_update_image_build.sh to build out the final update image.

        swu_generate_part_img.sh - generate partition mirror image
        -o specify output image name.
        -d directory that contains image files. These files will be copied into the image.
        -s image size. The size will be passed to truncate command directly.
        The size argument is an integer and optional unit.
        For example: 10K is 10*1024. Units are K,M,G,T,P,E,Z,Y, powers of 1024 or KB,MB,... powers of 1000.
        -f image filesystem format. vfat, ext2, ext3 and ext4 are supported.
        -h print this help.

        e.g. make boot partition

            swu_generate_part_img.sh -o slotb_boot_pt_120M.mirror -d your-target-boot-images-dir -s 120M -f vfat

            ls -al your-target-boot-images-dir
                Image
                imx8qm-cnh-pegasus.dtb
                dpfw.bin
                hdmirxfw.bin
                hdmitxfw.bin
                tee.bin

        then use swu_update_image_build.sh to make update image.

        swu_update_image_build.sh - generate update image
        -o specify output image name. Current default is <SOC_NAME>_<CONTAINER_VER>_<slot>_<BSP_VER>_<COPY_MODE>_<STORAGE_DEVICE>_<date>.
        -d enable double slot copy. default is single slot copy.
        -e enable emmc. default is sd.
        -s Specify public key file for sign image generation.
        -b soc name. Currently, imx93, imx8mm and imx6ull are supported.
        -g Compress image with gzip. Note that if installed-directly = true; is not specified in sw-description, compressed package need to be decompressed in RAM. Make sure ram is enough to hold the image.
        -h print this help.

        single copy :  swu_update_image_build.sh  -e -s priv.pem -b imx8qm -g
        double copy :  swu_update_image_build.sh  -d -e -s priv.pem -b imx8qm -g

        the output image is e.g. imx8qm_1.0_LF_v5.15.71_2.2.1_singlecopy_emmc_image_20231226_sign.swu  or
                                 imx8qm_1.0_LF_v5.15.71_2.2.1_doublecopy_emmc_image_20231226_sign.swu

```

</br>
</br>　


#### Flush base image, Upgrade update image

1. Flush the base image to emmc by uuu

```
    // single copy
    uuu -b emmc_all imx-boot-imx8qxpc0mek-sd.bin-flash_spl swu_singlecopy_rescue_imx8qm_emmc_20231226.sdcard

    // double copy
    uuu -b emmc_all imx-boot-imx8qxpc0mek-sd.bin-flash_spl swu_doublecopy_rescue_imx8qm_emmc_20231226.sdcard
```

2. Upgrade：single copy

```
    #. normal boot, then execute commands :

        fw_setenv bootslot singlerescue
        fw_setenv updrage_available 1

        then, reboot devices :

        reboot

    #. will boot to swupdate rescue ramdisk system.

        in the console, you will see "swuboot ramdisk" printed in the uboot log,

            Fastboot: Normal
            Normal Boot
            Hit any key to stop autoboot:  0
            swuboot ramdisk

   #. then, you can use Mongoose web interface or use swupdate CLI to Flush update image.

      - Mongoose：

            . open browser in your desktop PC, type http://YOUR_TARGET_DEVICE_IP:8080/
            . follow the web UI to select your update image, e.g. imx8qm_1.0_LF_v5.15.71_2.2.1_singlecopy_emmc_image_20231226_sign.swu
            . then upgrade will start to go.


      - swupdate CLI：

            . use scp or other method to put update image into your target device,
                e.g. put in /tmp/imx8qm_1.0_LF_v5.15.71_2.2.1_singlecopy_emmc_image_20231226_sign.swu.

            . execute commad,
                swupdate -v -i /tmp/imx8qm_1.0_LF_v5.15.71_2.2.1_doublecopy_emmc_image_20231218_sign.swu





```


3. Upgrade：double copy

```
    #. normal boot, either slot a or slot b is fine.

    #. then, you can use Mongoose web interface or use swupdate CLI to flush upgrade image. That's all the same as single copy as the above description.
```



</br>
</br>

---

</br>

- [appendix：swupdate image sing key](./appendix%EF%BC%9Aswupdate%20image%20sing%20key.md)