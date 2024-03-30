# Guide to Getting Started with Firecracker on Raspberry Pi

### Host specification
- Raspberry Pi 4 Model B(8GB)
- Ubuntu 22.04.4
- 5.15.0-1049-raspi

### How-to

First of all, you need to check KVM module is enabled.

```
sudo dmesg | grep -i kvm
```

or you can use
```
sudo apt install -y cpu-checker
kvm-ok
```

If you don't have KVM module, you have to install KVM.

Next, download firecracker binary.
```
wget https://github.com/firecracker-microvm/firecracker/releases/download/v1.1.0/firecracker-v1.1.0-aarch64.tgz
tar -zxvf firecracker-v1.1.0-aarch64.tgz
cp release-v1.1.0-aarch64/firecracker-v1.1.0-aarch64 firecracker
```

Then, you will need an uncompressed Linux kernel binary, and an ext4 file system image (to use as rootfs).
```
ARCH="$(uname -m)"

# linux kernel binary
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.6/${ARCH}/vmlinux-5.10.198

# rootfs
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.6/${ARCH}/ubuntu-22.04.ext4

# ssh key for rootfs
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.6/${ARCH}/ubuntu-22.04.id_rsa

# Set user read permission on the ssh key
chmod 400 ./ubuntu-22.04.id_rsa
```

Default size of rootfs is too small, so you have to resize to use bigger one.
```
truncate -s 5G ubuntu-22.04.ext4
e2fsck -f ubuntu-22.04.ext4
resize2fs ubuntu-22.04.ext4
```

Now, we can run firecracker!

I will explain two methods. (using **API requests(standard)** and using **config file**)

### Using API requests

We need two shells.

In the first shell, run firecracker binary.

```
rm -f /tmp/firecracker.socket
./firecracker --api-sock /tmp/firecracker.socket

# Result >
# Your prompt will be blocked...
```

In the second shell, communicate with firecracker process via HTTP requests.

Set the guest kernel.
```
arch=`uname -m`
kernel_path="vmlinux-5.10.198" 

curl --unix-socket /tmp/firecracker.socket -i \
      -X PUT 'http://localhost/boot-source'   \
      -H 'Accept: application/json'           \
      -H 'Content-Type: application/json'     \
      -d "{
            \"kernel_image_path\": \"${kernel_path}\",
            \"boot_args\": \"keep_bootcon console=ttyS0 reboot=k panic=1 pci=off\"
       }"
```

Set the guest rootfs.
```
rootfs_path="ubuntu-22.04.ext4"
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/drives/rootfs' \
  -H 'Accept: application/json'           \
  -H 'Content-Type: application/json'     \
  -d "{
        \"drive_id\": \"rootfs\",
        \"path_on_host\": \"${rootfs_path}\",
        \"is_root_device\": true,
        \"is_read_only\": false
   }"
```

Set the guest resources.
```
curl --unix-socket /tmp/firecracker.socket -i  \
  -X PUT 'http://localhost/machine-config' \
  -H 'Accept: application/json'            \
  -H 'Content-Type: application/json'      \
  -d '{
      "vcpu_count": 2,
      "mem_size_mib": 1024
  }'
```

Network setup.
```
sudo ip tuntap add tap0 mode tap
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward" # 포트 포워딩 활성
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i tap0 -o eth0 -j ACCEPT # 패킷이 tap0에서 eth0로 전달되는 것 허용

curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/network-interfaces/eth0' \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
      "iface_id": "eth0",
      "guest_mac": "AA:FC:00:00:00:01",
      "host_dev_name": "tap0"
    }'
```

Start microVM.
```
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/actions'       \
  -H  'Accept: application/json'          \
  -H  'Content-Type: application/json'    \
  -d '{
      "action_type": "InstanceStart"
   }'
```
Going back to your first shell, you should now see guest VM's prompt.

We have to finish network setup **in the guest**.
```
# 3줄은 매번 해줘야 -> 안하는 방법?
ip addr add 172.16.0.2/24 dev eth0
ip link set eth0 up
ip route add default via 172.16.0.1 dev eth0
echo nameserver 155.230.10.2 > /etc/resolv.conf
```

You can check `ping google.com` successfully work!

But if you run `apt update`, you will meet error.
```
   Reading package lists... Error!
E: flAbsPath on /var/lib/dpkg/status failed - realpath (2: No such file or directory)
E: Could not open file  - open (2: No such file or directory)
E: Problem opening
E: The package lists or status file could not be parsed or opened.
```

You can fix this by
```
mkdir /var/lib/dpkg
touch /var/lib/dpkg/status
```

### Using configuration file

```
./firecracker --api-sock /tmp/firecracker.socket --config-file vm_config.json
```

You can check the contents of `vm_config.json`

```
{
  "boot-source": {
    "kernel_image_path": "vmlinux-5.10.198",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off",
    "initrd_path": null
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "ubuntu-22.04.ext4",
      "is_root_device": true,
      "partuuid": null,
      "is_read_only": false,
      "cache_type": "Unsafe",
      "io_engine": "Sync",
      "rate_limiter": null
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 1024,
    "smt": false,
    "track_dirty_pages": false
  },
  "balloon": null,
  "network-interfaces": [
    {
      "iface_id": "eth0",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "tap0"
    }
  ],
  "vsock": null,
  "logger": null,
  "metrics": null,
  "mmds-config": null
}
```

If you want to power-off the guest VM, run `reboot` in the guest terminal.

You can use ssh to enter guest VM.
```
ssh -i ./ubuntu-22.04.id_rsa root@172.16.0.2
```

**Advanced:** If you are running multiple Firecracker MicroVMs in parallel, or have something else on your system using tap0 then you need to create a tap for each one, with a unique name.

**Advanced:** You also need to do the iptables set up for each new tap. If you have iptables rules you care about on your host, you may want to save those rules before starting.

### References
- https://github.com/firecracker-microvm/firecracker/blob/v1.1.0/docs/getting-started.md
- https://github.com/firecracker-microvm/firecracker/blob/v1.1.0/docs/network-setup.md
- https://dev.to/l1x/getting-started-with-firecracker-on-raspberry-pi-1pbc
- https://s8sg.medium.com/quick-start-with-firecracker-and-firectl-in-ubuntu-f58aeedae04b

### TODO
- [ ] Whenever boot guest VM, I have to do network setup in the guest. Is there any solution to do this only once?
