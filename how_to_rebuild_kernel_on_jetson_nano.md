# How to rebuild the kernel to activate KVM and GIC on the NVIDIA Jetson Nano

original reference: https://github.com/lattice0/jetson_nano_kvm

<pre><code>uname -r

Result >
4.9.253-tegra</code></pre>

We can check that KVM and GIC are not enabled.

<pre><code>sudo dmesg | grep -i kvm

Result >
nothing</code></pre>

<pre><code></code>sudo dmesg | grep -i gic

Result ><br>[    1.041740] gic 702f9000.agic: GIC IRQ controller registered</code></pre>

<pre><code>ls /proc/device-tree/interrupt-controller

Result >
 compatible          interrupt-controller   linux,phandle   phandle   status
'#interrupt-cells'   interrupt-parent       name            reg</code></pre>

![스크린샷 2024-04-13 194753](https://github.com/yunhachoi/manual/assets/161846673/7707f9ba-6a85-452f-83ef-cd43305fcfc9)


First, install packages.

<pre><code>sudo apt update && sudo apt install -y build-essential bc git curl wget xxd kmod libssl-dev</code></pre>

You should download the latest Jetson Linux kernel source from [here](https://developer.nvidia.com/embedded/jetson-linux-archive) that supports your board.

<pre><code>wget https://developer.nvidia.com/downloads/embedded/l4t/r32_release_v7.4/sources/t210/public_sources.tbz2
tar -jxvf public_sources.tbz2
JETSON_NANO_KERNEL_SOURCE=~/Linux_for_Tegra/source/public/
cd $JETSON_NANO_KERNEL_SOURCE
tar -jxvf kernel_src.tbz2</code></pre>

Copy uploaded `tegra_defconfig` to `${JETSON_NANO_KERNEL_SOURCE}/kernel/kernel-4.9/arch/arm64/configs/`

Important configurations.

<pre>
CONFIG_HAVE_KVM_IRQCHIP=y
CONFIG_HAVE_KVM_IRQFD=y
CONFIG_HAVE_KVM_IRQ_ROUTING=y
CONFIG_HAVE_KVM_EVENTFD=y
CONFIG_KVM_MMIO=y
CONFIG_HAVE_KVM_MSI=y
CONFIG_HAVE_KVM_CPU_RELAX_INTERCEPT=y
CONFIG_KVM_VFIO=y
CONFIG_HAVE_KVM_ARCH_TLB_FLUSH_ALL=y
CONFIG_KVM_GENERIC_DIRTYLOG_READ_PROTECT=y
CONFIG_KVM_COMPAT=y
CONFIG_VIRTUALIZATION=y
CONFIG_KVM_ARM_VGIC_V3_ITS=y
CONFIG_KVM=y
CONFIG_KVM_ARM_HOST=y
CONFIG_KVM_ARM_PMU=y
CONFIG_VHOST_NET=m
CONFIG_VHOST=m
 
CONFIG_IRQCHIP=y
CONFIG_ARM_GIC=y
CONFIG_FIQ=y
CONFIG_ARM_GIC_PM=y
CONFIG_ARM_GIC_MAX_NR=1
CONFIG_ARM_GIC_V2M=y
CONFIG_ARM_GIC_V3=y
CONFIG_ARM_GIC_V3_ITS=y</pre>

Compiling the kernel now would already activate KVM,

but we would still miss an important feature that makes virtualization much faster: `irq chip`

Without it, virtualization is still possible but an emulated irq chip is much slower. On firecracker (a virtualization tool written by AWS), it will not work as it requires this.

What we need to do is specify, in the device tree, the features of the irq chip on the CPU. The device tree is a file that contains addresses for all devices on the Jetson Nano chip.

Apply the below patch.

<pre><code>cd ${JETSON_NANO_KERNEL_SOURCE}/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc
vim tegra210-soc-base.dtsi</code></pre>

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
-               status = "disabled";
+               status = "okay";
        };
```

Start kernel compile.
 
<pre><code>JETSON_NANO_KERNEL_SOURCE=~/Linux_for_Tegra/source/public
TEGRA_KERNEL_OUT=$JETSON_NANO_KERNEL_SOURCE/build
KERNEL_MODULES_OUT=$JETSON_NANO_KERNEL_SOURCE/modules
cd $JETSON_NANO_KERNEL_SOURCE
  
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra tegra_defconfig
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target zImage
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target modules
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target dtbs
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra INSTALL_MOD_PATH=$KERNEL_MODULES_OUT modules_install</code></pre>

Make a backup.

<pre><code>sudo cp -r /boot /boot_original
sudo cp -r /lib /lib_original</code></pre>

Override drivers, modules, and image.

<pre><code>cd $JETSON_NANO_KERNEL_SOURCE/modules/lib/
sudo cp -r firmware/* /lib/firmware
sudo cp -r modules/* /lib/modules</code></pre>


<pre><code>cd $JETSON_NANO_KERNEL_SOURCE/build/arch/arm64/
sudo rsync -avh boot/* /boot</code></pre>

Modify the config file. (Add `FDT` label)
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

You have to `reboot`.

Now, we can check that kernel version has changed.
```
uname -r

Result >
4.9.337-tegra
```

KVM module is enabled.
```
sudo dmesg | grep -i kvm

Result >
[    1.050075] kvm [1]: 8-bit VMID
[    1.050081] kvm [1]: IDMAP page: 84f8d000
[    1.050085] kvm [1]: HYP VA range: 4000000000:7fffffffff
[    1.052403] kvm [1]: Hyp mode initialized successfully
[    1.052474] kvm [1]: vgic-v2@50044000
[    1.052638] kvm [1]: vgic interrupt IRQ1
[    1.052662] kvm [1]: virtual timer IRQ8
```

Also, GIC is enabled.
```
sudo dmesg | grep -i gic

Result > 
[    0.000000] GIC: Using split EOI/Deactivate mode
[    1.053149] kvm [1]: vgic-v2@50044000
[    1.053319] kvm [1]: vgic interrupt IRQ1
[    1.076684] gic 702f9000.agic: GIC IRQ controller registered
```

See that the node **interrupts**, which didn't exist before, was added.

This means that the irq interrupt activation worked.
```
ls /proc/device-tree/interrupt-controller

Result >
 compatible          interrupt-controller   interrupts      name      reg
'#interrupt-cells'   interrupt-parent       linux,phandle   phandle   status
```

![image](https://github.com/yunhachoi/manual/assets/161846673/dddf1a45-b108-4fbf-bcb8-a52f43ca495d)
