customizing the Xperia E1 kernel
--------------------------------

building the kernel
-------------------
This is the easy part.

*   Download the source tarball from Sony's site. I used [this one](http://developer.sonymobile.com/downloads/xperia-open-source-archives/open-source-archive-for-20-1-a-2-13-20-1-b-2-15-and-20-1-b-2-16/).

*   Download a prebuilt toolchain:

        git clone --depth=1 https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.7/

*   Build the kernel:

        cd kernel
        mkdir ../out
        ARCH=arm CROSS_COMPILE=/path/to/toolchain/arm-eabi-4.7/bin/arm-eabi- make O=../out msm8610_build_defconfig zImage -j2

    ... and after a while you should have a brand new zImage under `../out/arch/arm/boot/zImage`

getting the damn thing into the phone
-------------------------------------
Now this is where things get messy. You need to build an image of the boot partition.

Traditionally, on Android this was just a header followed by the zImage and the ramdisk. You'd use a python script called `mkbootimg` to build it (clone [this](https://android.googlesource.com/platform/system/core) if curious). However on ARM you're now required to use a [dtb blob](http://elinux.org/Device_Tree) (aka Flattened Device Tree). In the case of Xperia phones (or the Xperia E1 at least), following the ramdisk there's a structure with a signature that starts with `QCDT` (presumably QC stands for Qualcomm) that contains a bunch of dtb images. See [this](https://raw.githubusercontent.com/sonyxperiadev/mkqcdtbootimg/master/dtbtool.txt) for details.

Now, Sony has an open source tool called [mkqcdtbootimg](https://github.com/sonyxperiadev/mkqcdtbootimg) that is supposed to put together a boot image with the dtb files that were built together with the kernel, but I didn't have much luck with that. So I [forked the tool](https://github.com/mpersano/mkqcdtbootimg) and added an option to make it use a qcdt that can be extracted from a prebuilt boot image.

So if you have an existing boot image, you can extract the ramdisk and qcdt images (with [this tool](https://github.com/mpersano/bootimg-tools/blob/master/split-bootimage.py)) and use `mkqcdbootimg` to build a new boot image with your new kernel.

How do you get a boot image, though?

*If* you had root on your phone I suppose you could just do something like...

    dd if=/dev/block/platform/msm_sdcc.1/by-name/boot of=/data/boot.img

However you don't. So you may have to use a certain closed-source application (written in C#) that will get a bunch of files from Sony's servers; one of those files will be a gzipped zip file (yes really), containing another file with a name like kernel-BLAH-BLAH.sin; then you'd use another closed-source application (this time written in Java) that will extract the ramdisk and qcdt images from this file.

I'll update this section once I figure out a less roundabout way of doing this.

Anyway, once you have the qcdt and ramdisk files, you can build a boot.img with an incantation like the following:

    ./mkqcdtbootimg --kernel zImage --ramdisk ramdisk.gz --qcdt qcdt --base 0x00000000 --ramdisk_offset 0x2008000 --kernel_offset 0x10000 --tags_offset 0x1e08000 --pagesize 2048 --cmdline "androidboot.hardware=qcom user_debug=31 maxcpus=2 msm_rtb.filter=0x3F ehci-hcd.park=3 msm_rtb.enable=0 lpj=192598 dwc3.maximum_speed=high dwc3_msm.prop_chg_detect=Y" -o boot.img

... which you can flash on your phone with fastboot.
