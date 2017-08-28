---
layout: post
title: raspberryPi 1/2/3 build new kernel
categories: raspberrypi bare-metal
---
Build Kernel for RaspberryPi

<!-- more -->

make the config first
=====================

### Pi 1


{% highlight bash %}
cd linux
KERNEL=kernel
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcmrpi_defconfig
{% endhighlight %}

### Pi 2/3
{% highlight bash %}

cd linux
KERNEL=kernel7
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
{% endhighlight %}

build zImage + modules
==============
Then for both:
{% highlight bash %}

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-  zImage modules dtbs
{% endhighlight%}

install with dtbs
=================
install the modules
{% highlight bash %}
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install

copy bcm2709-rpi-2-b.dtb
sudo cp arch/arm/boot/dts/*.dtb mnt/fat32/
copy overlays:
sudo cp arch/arm/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
{% endhighlight %}
