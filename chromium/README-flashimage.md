---
author: Stephane Desneux <sdx@iot.bzh>
title: Booting AGL images from SDCard
---

## Purpose

Here are the commands to run on a Linux host to create a bootable SDcard from a full image file and boot a Renesas R-Car Gen3 board (Starter Kit Pro / M3ULCB).

Requirements:

* bmaptools
* SDcard (>2GB) inserted and available at $DEVICE (/dev/sdX , replace X by appropriate letter - can be /dev/mmcblkX depending on reader device)
* the M3 board is preconfigured to boot on SDCard

#### TLDR; quick instructions

* Using wget or any other tool, download the raw image (named \*.wic.xz) and the associated bmap file (\*.bmap) from [this folder](images)
    ```bash
    wget http://...../*.wic.xz
    wget http://...../*.wic.bmap
    ```
* Find your device and set DEVICE variable:
    ```bash
    $ lsblk -dli -o NAME,TYPE,HOTPLUG | grep "disk\s+1$"
    sdk  disk       1    # <= use /dev/sdk
    $ DEVICE=/dev/sdX
    ```
* Run *bmaptool*:

    ```bash
    sudo bmaptool copy *.wic.xz $DEVICE
    ```
* Eject SDCard, insert in M3 board and turn it on.
* Enjoy! (if your firmware and uboot config are correct - see below)

## Install bmap-tools (recommended)

Bmap-tools is a generic tool for creating the block map (bmap) from a sparse file and copy a raw image using the block map. The idea is that large files, like raw system image files, can be copied or flashed a lot faster and more reliably with bmaptool than with traditional tools, like "dd" or "cp".

Bmap-tools sources are available on [github:intel/bmap-tools](https://github.com/intel/bmap-tools).
[Full documentation](https://source.tizen.org/documentation/reference/bmaptool) is also available (a bit old, but still relevant).

**Note**: Even if Bmap-tools is not strictly required for operation, it's highly recommended. You can still skip this section if you do not wish to install bmap-tools or don't find any package for it.

### RPM-based distribution

Bmap-tools is available as a noarch package here: [bmap-tools-3.3-1.17.1.noarch.rpm](http://iot.bzh/download/public/tools/bmap-tools/bmap-tools-3.3-1.17.1.noarch.rpm)

For example, on Opensuse 42.X:

```bash
sudo zypper in http://iot.bzh/download/public/tools/bmap-tools/bmap-tools-3.3-1.17.1.noarch.rpm
```

### Debian-based distribution (inc. Ubuntu)

bmap-tool is available in Debian distribution (not tested).

```bash
sudo apt-get install bmap-tools
```

## Download AGL image and bmap file

Download the image and the associated bmap file:

* the raw image (*.wic.xz)
* the bmap file (*.wic.bmap)

## Write a SDcard

1. Insert a SDcard (minimum 2GB)

2. Find the removable device for your card:

    The following commands which lists all removable disks can help to find the information:

    ```bash
    $ lsblk -dli -o NAME,TYPE,HOTPLUG | grep "disk\s+1$"
    sdk  disk       1
    ```

    Here, the device we'll use is /dev/sdk.

    Alternatively, a look at the kernel log will help:

    ```bash
    $ dmesg | tail -50
    ...
    [710812.225836] sd 18:0:0:0: Attached scsi generic sg12 type 0
    [710812.441406] sd 18:0:0:0: [sdk] 31268864 512-byte logical blocks: (16.0 GB/14.9 GiB)
    [710812.442016] sd 18:0:0:0: [sdk] Write Protect is off
    [710812.442019] sd 18:0:0:0: [sdk] Mode Sense: 03 00 00 00
    [710812.442642] sd 18:0:0:0: [sdk] No Caching mode page found
    [710812.442644] sd 18:0:0:0: [sdk] Assuming drive cache: write through
    [710812.446794]  sdk: sdk1
    [710812.450905] sd 18:0:0:0: [sdk] Attached SCSI removable disk
    ...
    ```

    For the rest of these instructions, we assume that the variable $DEVICE contains the name of the device to write to (/dev/sd? or /dev/mmcblk?). Export the variable:

    ```bash
    export DEVICE=/dev/[replace-by-your-device-name]
    ```

3. If the card is mounted automatically, unmount it through desktop helper or directly wih the command line:

    ```bash
    sudo umount ${DEVICE}*
    ```

4. Write onto SDcard

    Using bmap-tools:

    ```bash
    $ sudo bmaptool copy *.wic.xz $DEVICE
    bmaptool: info: discovered bmap file 'XXXXXXXXX.wic.bmap'
    bmaptool: info: block map format version 2.0
    bmaptool: info: 524288 blocks of size 4096 (2.0 GiB), mapped 364283 blocks (1.4 GiB or 69.5%)
    bmaptool: info: copying image 'XXXXXXXX.wic.xz' to block device '/dev/sdk' using bmap file 'XXXXXXXX.wic.bmap'
    bmaptool: info: 100% copied
    bmaptool: info: synchronizing '/dev/sdk'
    bmaptool: info: copying time: 4m 26.9s, copying speed 5.3 MiB/sec
    ```

    Using standard dd command (more dangerous):

    ```bash
    xz -cd *.wic.xz | sudo dd of=$DEVICE bs=4M; sync
    ```

## Upgrade M3 to latest Firmware

Due to recent additions in AGL/Eel.rc5 to support the Kingfisher board, it's necessary to upgrade the M3 to a recent firmware.

The procedure to upgrade the M3 board is documented [on the M3SK page on eLinux.org](https://elinux.org/R-Car/Boards/M3SK#Flashing_firmware).

IoT.bzh has tested successfully the following procedure using ''minicom'':

1. Download each files from the [firmware directory](firmware) and put them in a directory of your choice
2. Open a terminal
3. 'cd' to the directory chosen in 1.
3. run 'minicom -b 115200 -D /dev/ttyUSB0' (adjust the serial port device if needed, add permissions on device if needed)
4. **BE REALLY CAREFULL ON THIS STEP**: Follow instructions from [eLinux.org for M3SK](https://elinux.org/R-Car/Boards/M3SK#Flashing_firmware)
5. Reboot the board

After a successful flashing, the following versions (or later) should be available on the console boot log:

```
[ 0.000193] NOTICE: BL2: R-Car Gen3 Initial Program Loader(CA57) Rev.1.0.16
[ 0.005757] NOTICE: BL2: PRR is R-Car M3 Ver1.0
[ 0.010339] NOTICE: BL2: Board is Starter Kit Rev1.0
[ 0.015366] NOTICE: BL2: Boot device is HyperFlash(80MHz)
[ 0.020792] NOTICE: BL2: LCM state is CM
[ 0.024833] NOTICE: BL2: AVS setting succeeded. DVFS_SetVID=0x53
[ 0.030814] NOTICE: BL2: DDR3200(rev.0.27)NOTICE: [COLD_BOOT]NOTICE: ..0
[ 0.054032] NOTICE: BL2: DRAM Split is 2ch
[ 0.057917] NOTICE: BL2: QoS is default setting(rev.0.19)
[ 0.063415] NOTICE: BL2: Lossy Decomp areas
[ 0.067594] NOTICE: Entry 0: DCMPAREACRAx:0x80000540 DCMPAREACRBx:0x570
[ 0.074679] NOTICE: Entry 1: DCMPAREACRAx:0x40000000 DCMPAREACRBx:0x0
[ 0.081591] NOTICE: Entry 2: DCMPAREACRAx:0x20000000 DCMPAREACRBx:0x0
[ 0.088506] NOTICE: BL2: v1.3(release):b330e0e
[ 0.092995] NOTICE: BL2: Built : 08:53:12, Dec 22 2017
[ 0.098183] NOTICE: BL2: Normal boot
[ 0.101824] NOTICE: BL2: dst=0xe631f208 src=0x8180000 len=512(0x200)
[ 0.108373] NOTICE: BL2: dst=0x43f00000 src=0x8180400 len=6144(0x1800)
[ 0.114832] NOTICE: BL2: dst=0x44000000 src=0x81c0000 len=65536(0x10000)
[ 0.122057] NOTICE: BL2: dst=0x44100000 src=0x8200000 len=524288(0x80000)
[ 0.132536] NOTICE: BL2: dst=0x50000000 src=0x8640000 len=1048576(0x100000)

U-Boot 2015.04 (Dec 22 2017 - 09:53:06)

CPU: Renesas Electronics R8A7796 rev 1.0 
Board: M3ULCB 
I2C: ready 
DRAM: 1.9 GiB 

Flash: 64 MiB 

MMC: sh-sdhi: 0, sh-sdhi: 1 
In: serial 
Out: serial 
Err: serial 
...
```

Next step is to configure uboot properly.

## Configure M3 board for boot on SDcard

If not already done, you'll have to configure Uboot parameters.

1. Connect serial console on M3 board and start a terminal emulator on the USB serial port.
    Here, we use 'screen' on device /dev/ttyUSB0 but you could use any terminal emulator able to open the serial port at 115200 bauds (minicom , picocom ...)

    ```bash
    screen /dev/ttyUSB0 115200
    ```

2. Power up the board

3. Break at uboot prompt (press any key)

4. Set the following uboot variables:

    **WARNING: don't make a big copy/paste or some garbage characters may be sent to the console (issue with usb/serial port buffering?). Instead, copy one or two lines at a time.**

    ```uboot
#include(README-uboot-config.mdinc)
    ```

    Then save environment in NV flash:

    ```uboot
    saveenv
    ```

## Boot the board

At uboot prompt, type:

    ```
    run bootcmd
    ```

Alternatively, simply reset the board.

**NOTE**: Due to initial operations, first AGL boot can take longer (a few mintutes) than next ones.
