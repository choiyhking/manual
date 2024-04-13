# Enable KVM module, GIC on Jetson Nano

How to rebuild the kernel to activate KVM and GIC on the Nvidia Jetson Nano

original reference: https://github.com/lattice0/jetson_nano_kvm

We can check that KVM and GIC are not enabled.

<pre><code>sudo dmesg | grep -i kvm

> nothing</code></pre>

<pre><code></code>sudo dmesg | grep -i gic

Result><br>[    1.041740] gic 702f9000.agic: GIC IRQ controller registered</code></pre>

<pre><code>ls /proc/device-tree/interrupt-controller

Result>
 compatible          interrupt-controller   linux,phandle   phandle   status
  
'#interrupt-cells'   interrupt-parent       name            reg</code></pre>

First, install packages.

<pre><code>sudo apt update && sudo apt-get install -y build-essential bc git curl wget xxd kmod libssl-dev</code></pre>

You should download the latest Jetson linux kernel source from [here](https://developer.nvidia.com/embedded/jetson-linux-archive) that supports your board.

<pre><code>wget https://developer.nvidia.com/downloads/embedded/l4t/r32_release_v7.4/sources/t210/public_sources.tbz2
tar -jxvf public_sources.tbz2
JETSON_NANO_KERNEL_SOURCE=~/Linux_for_Tegra/source/public/
cd $JETSON_NANO_KERNEL_SOURCE
tar -jxvf kernel_src.tbz2</code></pre>


<pre><code>cd ${JETSON_NANO_KERNEL_SOURCE}/kernel/kernel-4.9
echo "CONFIG_KVM=y
CONFIG_VHOST_NET=m" >> arch/arm64/configs/tegra_defconfig</code></pre>

or use uploaded `tegra_defconfig`.

<pre><code>cd ${JETSON_NANO_KERNEL_SOURCE}/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc
vim tegra210-soc-base.dtsi</code></pre>

Apply the below patch.
```
--- a/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc/tegra210-soc-base.dtsi     2020-08-31 08:40:36.602176618 +0800
+++ b/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc/tegra210-soc-base.dtsi     2020-08-31 08:41:45.223679918 +0800
@@ -351,7 +351,10 @@
                #interrupt-cells = <3>;
                interrupt-controller;
                reg = <0x0 0x50041000 0x0 0x1000
-                      0x0 0x50042000 0x0 0x0100>;
+                       0x0 0x50042000 0x0 0x2000
+                       0x0 0x50044000 0x0 0x2000
+                       0x0 0x50046000 0x0 0x2000>;
+               interrupts = <GIC_PPI 9 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_HIGH)>;
                status = "disabled";
        };
```
 
<pre><code>JETSON_NANO_KERNEL_SOURCE=~/Linux_for_Tegra/source/public
TEGRA_KERNEL_OUT=$JETSON_NANO_KERNEL_SOURCE/build
KERNEL_MODULES_OUT=$JETSON_NANO_KERNEL_SOURCE/modules
cd $JETSON_NANO_KERNEL_SOURCE
  
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra tegra_defconfig
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target zImage
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target modules
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target dtbs
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra INSTALL_MOD_PATH=$KERNEL_MODULES_OUT modules_install</code></pre>

<pre><code>sudo cp -r /boot /boot_original
sudo cp -r /lib /lib_original</code></pre>


<pre><code>cd $JETSON_NANO_KERNEL_SOURCE/modules/lib/
sudo cp -r firmware/* /lib/firmware
sudo cp -r modules/* /lib/modules</code></pre>


<pre><code>cd $JETSON_NANO_KERNEL_SOURCE/build/arch/arm64/
sudo rsync -avh boot/* /boot</code></pre>

Now, KVM module exists.

Let's enable GIC.
Modify the config file. (Add FDT label)
```
sudo vim /boot/extlinux/extlinux.conf


TIMEOUT 30
DEFAULT primary

MENU TITLE L4T boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      FDT /boot/dts/tegra210-p3448-0000-p3449-0000-a00.dtb
      APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 loglevel=7 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0
```

```
sudo reboot
sudo dmesg | grep -i kvm

[    1.050075] kvm [1]: 8-bit VMID
[    1.050081] kvm [1]: IDMAP page: 84f8d000
[    1.050085] kvm [1]: HYP VA range: 4000000000:7fffffffff
[    1.052403] kvm [1]: Hyp mode initialized successfully
[    1.052474] kvm [1]: vgic-v2@50044000
[    1.052638] kvm [1]: vgic interrupt IRQ1
[    1.052662] kvm [1]: virtual timer IRQ8
```


Lastly, check the modified kernel version.
```
yunha@jetson:~$ sudo reboot
yunha@jetson:~$ uname -r
4.9.337-tegra
```
