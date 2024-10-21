# Guide to Getting Started with AWS Firecracker

## Host Specification
- Raspberry Pi 5
- Raspberry Pi OS (Debian GNU/Linux 12 (bookworm))
- 6.6.31+rpt-rpi-2712

## How-to
Download Kata Container source
```
wget https://github.com/kata-containers/kata-containers/releases/download/3.9.0/kata-static-3.9.0-arm64.tar.xz
sudo tar -xvf kata-static-3.9.0-arm64.tar.xz -C /

# default path is "/opt/kata"
# "kata-runtime" binary is utility program. (different from shimv2)
/opt/kata/bin/kata-runtime --version
# Result >
# kata-runtime  : 3.9.0
#    commit   : cdaaf708a18da8e5f7e2b9824fa3e43b524893a5
#    OCI specs: 1.1.0+dev
   
# Add symbolic link in order for containerd to reach these binaries from default system PATH
sudo ln -s /opt/kata/bin/kata-runtime /usr/local/bin
sudo ln -s /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin
sudo ln -s /opt/kata/bin/kata-collect-data.sh /usr/local/bin

# now, we can use without default path
kata-runtime --version
```
Check current host is available kata container
```
kata-runtime check
# Result >
# No newer release available
# ERRO[0000] Module is not loaded and it can not be inserted. Please consider running with sudo or as root  arch=arm64 module=vhost_vsock name=kata-runtime pid=4128 source=runtime
# ERRO[0000] kernel property vhost_vsock not found         arch=arm64 description="Host Support for Linux VM Sockets" name=vhost_vsock pid=4128 source=runtime type=module
# ERRO[0000] Module is not loaded and it can not be inserted. Please consider running with sudo or as root  arch=arm64 module=vhost name=kata-runtime pid=4128 source=runtime
# ERRO[0000] kernel property vhost not found               arch=arm64 description="Host kernel accelerator for virtio" name=vhost pid=4128 source=runtime type=module
# ERRO[0000] Module is not loaded and it can not be inserted. Please consider running with sudo or as root  arch=arm64 module=vhost_net name=kata-runtime pid=4128 source=runtime
# ERRO[0000] kernel property vhost_net not found           arch=arm64 description="Host kernel accelerator for virtio network" name=vhost_net pid=4128 source=runtime type=module
# ERRO[0000] ERROR: System is not capable of running Kata Containers  arch=arm64 name=kata-runtime pid=4128 source=runtime
# ERROR: System is not capable of running Kata Containers
```
some kernel modules aren't loaded which needed to run Kata Continaer

we need to rebuild kernel to load some modules!!
-> `vhost_vsock`, `vhost`, `vhost_net`

```
# install build dependencies
sudo apt update
sudo apt install -y vim git bc bison flex libssl-dev make libncurses5-dev

# download kernel source for the latest raspberry pi kernel
# "--depth=1" downloads the current active branch without any history
# omit this option to download the entire repository, including full history of all branches
# to download a different branch with no history
# git clone --depth=1 --branch <branch> https://github.com/raspberrypi/linux
git clone --depth=1 https://github.com/raspberrypi/linux

# build configuration
cd linux
KERNEL=kernel_2712
# "make bcm2711_defconfig" is
# to create .config file based on the `arch/arm64/configs/bcm2711_defconfig`

make menuconfig
##########################################################
# set the custom kernel build name 
#
# General setup > Local version - append to kernel release
# e.g., -v8-16k-virt

# set the vhost, vhost_net, vhost_vsock modules
#
# Device Drivers > VHOST drivers
# <M> Host kernel accelerator for virtio net
# <M> vhost virtio-vsock driver
##########################################################
```

## References
- https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md
- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md
- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-k8s-with-containerd-and-kata.md
- https://blog.niflheim.cc/posts/kata_containers_raspberry/
- https://blog.cloudkernels.net/posts/kata-fc-k3s-k8s/
- https://www.raspberrypi.com/documentation/computers/linux_kernel.html
- https://www.kernelconfig.io/index.html
