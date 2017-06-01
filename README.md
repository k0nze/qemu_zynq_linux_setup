# QEMU Zynq Petalinux setup on Linux Hosts

## Install QEMU

Download the QEMU source:
```bash
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
    -net nic -net user,hostfwd=tcp:127.0.0.1:10022-10.0.2.15:22
``` 

This will boot Petalinux on the _zc702_ you can either login directly in the QEMU terminal as user `root` or you can login via ssh with the following command:

```
ssh localhost -p 10022 -l root
```

## Filesharing

To easily share files between host and guest you can mount a directory of the host via ssh (requires `sshfs`) by excuting the following commands:
```
ssh -p 10022 localhost -l root "mkdir /media/sshfs"
sshfs -o port=10022,nonempty root@localhost:/media/sshfs <PATH_TO_MOUNT_DIR> 
```

This will create the `/media/sshfs` directory on the guest which contains all files from `<PATH_TO_MOUNT_DIR>`.
