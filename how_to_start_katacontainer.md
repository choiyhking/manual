# Guide to Getting Started with Kata Container

## Host Specification
- Raspberry Pi 5
- Raspberry Pi OS (Debian GNU/Linux 12 (bookworm))
- 6.6.31+rpt-rpi-2712

## How-to
Download Kata Container source.
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
Check if current host is available kata container.
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
Some kernel modules aren't loaded which needed to run Kata Continaer.

We need to rebuild kernel to load modules !!

-> `vhost_vsock`, `vhost`, `vhost_net`

**(I realized that we can load these modules without kernel rebuild...so you can skip and go to module loading section.)**

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
# Set the custom kernel build name 
#
# General setup > Local version - append to kernel release
# e.g., -v8-16k-virt

# Set the vhost, vhost_net, vhost_vsock modules
#
# Device Drivers > VHOST drivers
# <M> Host kernel accelerator for virtio net
# <M> vhost virtio-vsock driver
##########################################################
```

If you finished configuration, "SAVE" as `.config`.

```
# build kernel
# it takes time !!
make -j6 Image.gz modules dtbs

# install kernel modules onto the boot media
sudo make -j6 modules_install

# Then, install the kernel and Device Tree Blobs into the boot partition, backing up your original kernel.
# Run the following commands to create a backup image of the current kernel, 
# install the fresh kernel image, overlays, README, and unmount the partitions:
sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image.gz /boot/firmware/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/overlays/

sudo reboot
```

Check the changed kernel version.
```
uname -r
# Result >
# 6.6.56-v8-16k-virt+
```

Successfully changed !!

It's time to load modules.
```
lsmod | grep vhost
# Result > nothing

# we need to "load" the modules
sudo modprobe -v vhost_vsock
sudo modprobe -v vhost_net

lsmod | grep vhost
# Result >
# vhost_net              49152  0
# tun                    98304  1 vhost_net
# tap                    49152  1 vhost_net
# vhost_vsock            49152  0
# vmw_vsock_virtio_transport_common    81920  1 vhost_vsock
# vhost                  81920  2 vhost_vsock,vhost_net
# vhost_iotlb            49152  1 vhost
# vsock                  81920  2 vmw_vsock_virtio_transport_common,vhost_vsock
```
Now, kernel modules which we need are successfully loaded.

```
kata-runtime check
# Result >
# No newer release available
# System is capable of running Kata Containers
```

**We can simply use Kata Container with Docker.**

**(Docker does bothersome things, e.g., network and runtime configurations instead of us)**

```
sudo docker run --runtime io.containerd.kata.v2 hello-world
```

Otherwise, we have to do many things as follows.

First, install containerd.
```
sudo apt install -y containerd

containerd --version
# Result >
# containerd github.com/containerd/containerd 1.6.20~ds1 1.6.20~ds1-1+b1

runc --version
# Result >
# runc version 1.1.5+ds1
# commit: 1.1.5+ds1-1+deb12u1
# spec: 1.0.2-dev
# go: go1.19.8
# libseccomp: 2.5.4
```

```
# containerd.service: config file to manage containerd as service by systemd
# If we used apt install, it's automatically created

systemctl show -p FragmentPath containerd.service
# Result >
# FragmentPath=/lib/systemd/system/containerd.service
```

We have to modify containerd configuration to use kata runtime(i.e., `containerd-shim-kata-v2`) as a low-level container runtime(e.g., `runc`)
```
sudo vim /etc/containerd/config.toml
```
```
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/usr/lib/cni"
      conf_dir = "/etc/cni/net.d" 
 
    ###################################
    # configuration for Kata Containers
    [plugins."io.containerd.grpc.v1.cri".containerd]
      no_pivot = false
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
         privileged_without_host_devices = false
         runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
         runtime_type = "io.containerd.kata.v2"
         privileged_without_host_devices = true
         [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata.options]
            ConfigPath = "/opt/kata/share/defaults/kata-containers/configuration.toml"
     ###################################
     ###################################

  [plugins."io.containerd.internal.v1.opt"]
    path = "/var/lib/containerd/opt"
```

Apply the modified configurations.
```
sudo systemctl restart containerd.service
sudo systemctl status containerd.service
```

Test the Kata Container installation.
```
image="docker.io/library/busybox:latest"
sudo ctr image pull "$image"
sudo ctr run --runtime "io.containerd.kata.v2" --rm -t "$image" test-kata uname -r
# Result >
# 6.1.62 
```
![image](https://github.com/user-attachments/assets/bed44165-55c1-4d29-be55-24b3c6388c7b)

Different from host's kernel and runtime is "io.containerd.kata.v2" not runc.

Successfully configured !! 

But `ctr` is not user-friendly, incompatible with docker-like commands and lacks some functionalities.

We need to use `nerdctl`: Docker compatible CLI for containerd
```
wget https://github.com/containerd/nerdctl/releases/download/v1.7.7/nerdctl-1.7.7-linux-arm64.tar.gz
sudo tar Cxzvvf /usr/local/bin nerdctl-1.7.7-linux-arm64.tar.gz

sudo systemctl enable --now containerd
sudo nerdctl run --runtime io.containerd.kata.v2 -it ubuntu:22.04 /bin/bash
```

But it's not working.

![image](https://github.com/user-attachments/assets/96e44572-1857-4cee-9396-7567b58e9429)

It seems like problem about CNI plugin.
We need to install CNI plugin to configure network manually.

```
# install golang manually from the source
wget https://go.dev/dl/go1.23.2.linux-arm64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.23.2.linux-arm64.tar.gz
export PATH=$PATH:/usr/local/go/bin
source ~/.bashrc

go version
# Result > go version go1.23.2 linux/arm64

# Install CNI plugins
git clone https://github.com/containernetworking/plugins.git
pushd plugins
./build_linux.sh
sudo mkdir /opt/cni
sudo cp -r bin /opt/cni/
popd
```

Test again.
```
sudo nerdctl run --runtime io.containerd.kata.v2 -it ubuntu:22.04 /bin/bash
```
Kata container is created, but there's still WARNING about cgroup.

![image](https://github.com/user-attachments/assets/4158a1da-07f8-4ac3-a2cf-58fd46c235bd)

```
sudo vim /boot/firmware/cmdline.txt
# Add "systemd.unified_cgroup_hierarchy=0" at the end of the line.

# you have to write options in one line and seperate with blank
# 0 -> cgroup v1
# 1 -> cgroup v2

sudo vim /etc/containerd/config.toml
# Add "SystemdCgroup = false" to [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata.options]

sudo vim /opt/kata/share/defaults/kata-containers/configuration.toml
# Modify "sandbox_cgroup_only = true"

sudo reboot
sudo nerdctl run --runtime io.containerd.kata.v2 -it ubuntu:22.04 /bin/bash
```
![image](https://github.com/user-attachments/assets/a7ed7eec-4419-4463-8c76-45b2463ceaa2)

Finally success !! 

Also, you have to check network inside the container.(e.g., `apt update`, `ping`)

## References
- https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md
- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md
- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-k8s-with-containerd-and-kata.md
- https://blog.niflheim.cc/posts/kata_containers_raspberry/
- https://blog.cloudkernels.net/posts/kata-fc-k3s-k8s/
- https://www.raspberrypi.com/documentation/computers/linux_kernel.html
- https://www.kernelconfig.io/index.html
- https://github.com/containerd/nerdctl/releases/tag/v1.7.7
- https://github.com/containernetworking/plugins
