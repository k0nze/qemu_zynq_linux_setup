<!--
Author: Konstantin Lübeck (University of Tübingen, Chair for Embedded Systems)
-->

# QEMU Zynq Petalinux setup on Linux Hosts

## Install QEMU

Download the QEMU source:
```
git clone git://git.qemu-project.org/qemu.git
```

Get the `pixman` and `dtc` submodules:
```
cd qemu
git submodule update --init pixman dtc
```

Compile with:
```
./configure --target-list="aarch64-softmmu,microblazeel-softmmu" --enable-fdt --disable-kvm --disable-xen
make -j4
```

## Boot Petalinux

In order to boot petalinux you have to download the devicetree and the Petalinux images from Xilinx. Open the following URL: [http://www.wiki.xilinx.com/Zynq+Releases](http://www.wiki.xilinx.com/Zynq+Releases)

Download the latest release (here _Zynq 2016.4 Release_) for _zc702_ or download via command line with:
```
wget http://www.wiki.xilinx.com/file/view/2016.4-zc702-release.tar.xz/603943822/2016.4-zc702-release.tar.xz
```

Extract the archive:
```
tar xf 2016.4-zc702-release.tar.xz
``` 

You will find the following files in the extracted directory `zc702`:
```
BOOT.bin
devicetree.dtb
fsbl-zc702-zynq7.elf
u-boot.elf
uImage
uramdisk.image.gz
```

To boot Petalinux with QEMU execute the following command:
``` 
<PATH_TO_QEMU_GIT_REPO>/aarch64-softmmu/qemu-system-aarch64 \ 
    -M xilinx-zynq-a9 \
    -serial /dev/null \
    -serial mon:stdio \
    -display none \
    -kernel <PATH_TO_PETALINUX>/zc702/uImage \
    -dtb <PATH_TO_PETALINUX>/zc702/devicetree.dtb \
    --initrd <PATH_TO_PETALINUX>/zc702/uramdisk.image.gz \
    -net nic -net user,hostfwd=tcp:127.0.0.1:10022-10.0.2.15:22,tftp=<PATH_TO_HOST_DIR>
``` 

This will boot Petalinux on the _zc702_ you can either login directly in the QEMU terminal as user `root` or you can login via ssh with the following command:

```
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -p 10022 root@localhost
```

Since the Petalinux guest generates a new SSH key each time it boots the options `-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no` are used which disable the key check.

You can terminate qemu by pressing [CTRL+A] [X] in the qemu terminal.

## Filesharing

Petalinux was booted with the `-net` option `tftp=<PATH_TO_HOST_DIR>` which started a ftp server on the host with the root directory `<PATH_TO_HOST_DIR>`. 

You can retrieve files from this directory on the guest with the following command:
```
tftp -r <HOST_FILE> -g 10.0.2.2
```

Unfortunately the server is read only.

## Compile for Petalinux

To compile a C program on your host you need the `arm-linux-gnueabihf-gcc` compiler either you have a recent version of Xilinx Vivado SDx which provides this compiler or you have to install it manually for your host:

 * Ubuntu
    ```
    sudo apt-get install gcc-4.8-arm-linux-gnueabihf
    ```

 * TODO: Other Linux distros

You can find a [Hello World C program](./src/helloworld.c) in [src](./src) and a [Makefile](./src/Makefile) to compile it for Petalinux. Compile it with:
```
make
```

Copy the binary to your directory for `tftp`:
```
cp helloworld <PATH_TO_HOST_DIR>
```

On the guest system execute the following commands to run the Hello World program:
```
tftp -r helloworld -g 10.0.2.2
chmod +x helloworld
./helloworld
```

## Mounting a Virtual SD Card on the Host Computer and in Petalinux running on QEMU

To mount a virtual SD card on the host computer you have to install `nbd-client` (network block device client) and load its kernel module.

 * Ubuntu
    ```
    sudo apt-get install nbd-client
    sudo modprobe nbd
    ```
 * CentOS
    ```
    sudo yum install nbd
    #TODO load kernel module
    ```

Check if the kernel module was sucessfully loaded with:
```
lsmod | grep nbd
```

In the next step you have to create a `qcow2` image with `qemu-img`:
```
<PATH_TO_QEMU_GIT_REPO>/qemu-img create -f qcow2 <PATH_TO_QCOW2_IMG>/sdcard.qcow2 1000M
```

This will create a `qcow2` image with the size of 1GB. To mount the `sdcard.qcow2` on the host computer execute the following command to connect the `qcow2` to a network block device:
```
sudo <PATH_TO_QEMU_GIT_REPO>/qemu-nbd --connect=/dev/nbd0 <PATH_TO_QCOW2_IMG>/sdcard.qcow2
```

Check if the `qcow2` image was mounted correctly with:
```
sudo fdisk /dev/nbd0 -l
```

You should see something like this:
```
Disk /dev/nbd0: 1000 MiB, 1048576000 bytes, 2048000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

Now you can format the SD card image with `fdisk` executing:
```
sudo fdisk /dev/nbd0
```

 1. Press [n] to create a new partition.
 2. Choose [p] to make the new partition primary.
 3. Press [Enter] to accept the default `1`.
 4. Press [Enter] to accept the default `2048`.
 5. Press [Enter] to accept the default `2047999`.
 6. Press [w] to write the new partition table to the SD card image and leave `fdisk`.

Check if the partition table is correct with:
```
sudo fdisk /dev/nbd0 -l
```

You should see something like this:
```
Disk /dev/nbd0: 1000 MiB, 1048576000 bytes, 2048000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x6dde57fa

Device      Boot Start     End Sectors  Size Id Type
/dev/nbd0p1       2048 2047999 2045952  999M 83 Linux
```

After the partition table is written you can format the SD card image with vfat.:
```
sudo mkfs.vfat /dev/nbd0p1
```

Now you can mount the `/dev/nbd0p1` partition:
```
mkdir <MOUNT_POINT>
sudo mount -t vfat /dev/nbd0p1 <MOUNT_POINT>
sudo chmod 775 -R <MOUNT_POINT>
```

Now you can copy files to the SD card image. If you are done copying files unmount and disconnect it with:
```
sudo umount <MOUNT_POINT>
<PATH_TO_QEMU_GIT_REPO>/qemu-nbd --disconnect <PATH_TO_QCOW2_IMG>/sdcard.qcow2
```

Start QEMU with the following command:

```
<PATH_TO_QEMU_GIT_REPO>/aarch64-softmmu/qemu-system-aarch64 \ 
    -M xilinx-zynq-a9 \
    -serial /dev/null \
    -serial mon:stdio \
    -display none \
    -kernel <PATH_TO_PETALINUX>/zc702/uImage \
    -dtb <PATH_TO_PETALINUX>/zc702/devicetree.dtb \
    --initrd <PATH_TO_PETALINUX>/zc702/uramdisk.image.gz \
    -net nic -net user,hostfwd=tcp:127.0.0.1:10022-10.0.2.15:22,tftp=<PATH_TO_HOST_DIR>
	-drive file=<PATH_TO_QCOW2_IMG>/sdcard.qcow2,if=sd,index=0,media=disk
```

Login into Petalinux as `root` and check if the SD card is visible with:
```
fdisk -l
```

You should see something like this:
```
Disk /dev/mmcblk0: 1048 MB, 1048576000 bytes
123 heads, 59 sectors/track, 282 cylinders
Units = cylinders of 7257 * 512 = 3715584 bytes

        Device Boot      Start         End      Blocks  Id System
/dev/mmcblk0p1               1         283     1022976  83 Linux
Partition 1 has different physical/logical beginnings (non-Linux?):
     phys=(0, 32, 33) logical=(0, 34, 43)
Partition 1 has different physical/logical endings:
     phys=(127, 122, 59) logical=(282, 25, 51)
```

Mount the SD card under Petalinux with:
```
mkdir <MOUNT_POINT>
mount -t vfat /dev/mmcblk0p1 <MOUNT_POINT>
```
