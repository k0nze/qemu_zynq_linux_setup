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


