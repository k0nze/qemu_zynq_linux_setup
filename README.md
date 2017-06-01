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

unfortunately the server is read only.

## Compile for Petalinux

You can find a Hello World C program in [src](./src) and a Makefile to compile it for Petalinux. Compile it with:
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

