# Guide to Getting Started with Firecracker on Raspberry Pi

### Host specification
- Raspberry Pi 5
- Debian GNU/Linux 12 (bookworm)
- 6.6.31+rpt-rpi-2712

### How-to
Firecracker requires read/write access to `/dev/kvm` exposed by the KVM module.

First of all, you need to check KVM module is exist.
```
sudo dmesg | grep -i kvm

Result >
[    0.051848] kvm [1]: IPA Size Limit: 40 bits
[    0.051872] kvm [1]: GICV region size/alignment is unsafe, using trapping (reduced performance)
[    0.051900] kvm [1]: vgic interrupt IRQ9
[    0.051913] kvm [1]: VHE mode initialized successfully
```

or you can use
```
sudo apt install -y cpu-checker
kvm-ok
```

If you don't have KVM module, you have to load it.

Some Linux distributions use the `kvm` group to manage access to /dev/kvm, while others rely on access control lists. 

If you have the ACL package for your distro installed, you can grant Read+Write access with:
```
sudo setfacl -m u:${USER}:rw /dev/kvm

# check status
sudo getfacl /dev/kvm

Result >
getfacl: Removing leading '/' from absolute path names
# file: dev/kvm
# owner: root
# group: kvm
user::rw-
user:pi:rw-
group::rw-
mask::rw-
other::---
```

You can check if you have access to `/dev/kvm` with:
```
[ -r /dev/kvm ] && [ -w /dev/kvm ] && echo "OK" || echo "FAIL"

Result > 
OK
```

Next, download firecracker binary.
```
ARCH="$(uname -m)" 
release_url="https://github.com/firecracker-microvm/firecracker/releases"
latest=$(basename $(curl -fsSLI -o /dev/null -w  %{url_effective} ${release_url}/latest))
curl -L ${release_url}/download/${latest}/firecracker-${latest}-${ARCH}.tgz \
| tar -xz

# echo $ARCH
# Result > aarch64
# echo $latest
# Result > v.1.9.1

# Rename the binary to "firecracker"
mv release-${latest}-$(uname -m)/firecracker-${latest}-${ARCH} firecracker
```

Then, you will need an uncompressed Linux kernel binary, and an ext4 file system image (to use as rootfs).
```
ARCH="$(uname -m)"

latest=$(wget "http://spec.ccfc.min.s3.amazonaws.com/?prefix=firecracker-ci/v1.10/aarch64/vmlinux-6.1&list-type=2" -O - 2>/dev/null | grep "(?<=<Key>)(firecracker-ci/v1.10/aarch64/vmlinux-6\.1\.[0-9]{3})(?=</Key>)" -o -P)

# echo $latest
# Result > firecracker-ci/v1.10/aarch64/vmlinux-6.1.102

# Download a linux kernel binary
wget "https://s3.amazonaws.com/spec.ccfc.min/${latest}"

# Download a rootfs
wget "https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.10/${ARCH}/ubuntu-22.04.ext4"

# Download the ssh key for the rootfs
wget "https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.10/${ARCH}/ubuntu-22.04.id_rsa"

# Set user read permission on the ssh key
chmod 400 ./ubuntu-22.04.id_rsa
```

Default size of file system image is too small, so you have to resize to use bigger one.
```
truncate -s 5G ubuntu-22.04.ext4
e2fsck -f ubuntu-22.04.ext4
resize2fs ubuntu-22.04.ext4
```

Now, we can run firecracker!

There are two methods. (using **API requests** or **configuration file**)

### Using API requests

We need two shells.

In the first shell, run firecracker binary.

```
API_SOCKET="/tmp/firecracker.socket"

# Remove API unix socket
sudo rm -f $API_SOCKET

# Run firecracker
sudo ./firecracker --api-sock "${API_SOCKET}"

Result >
2024-10-20T15:00:36.697396842 [anonymous-instance:main] Running Firecracker v1.9.1
(and your shell will be blocked...)
```

In the second shell, communicate with firecracker process via HTTP requests.
```
TAP_DEV="tap0"
TAP_IP="172.16.0.1"
MASK_SHORT="/30"

# Setup network interface
sudo ip link del "$TAP_DEV" 2> /dev/null || true
sudo ip tuntap add dev "$TAP_DEV" mode tap
sudo ip addr add "${TAP_IP}${MASK_SHORT}" dev "$TAP_DEV"
sudo ip link set dev "$TAP_DEV" up

# Enable ip forwarding
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"

HOST_IFACE="wlan0"

# Set up microVM internet access
sudo iptables -t nat -D POSTROUTING -o "$HOST_IFACE" -j MASQUERADE || true
sudo iptables -D FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT \
    || true
sudo iptables -D FORWARD -i "$TAP_DEV" -o "$HOST_IFACE" -j ACCEPT || true
sudo iptables -t nat -A POSTROUTING -o "$HOST_IFACE" -j MASQUERADE
sudo iptables -I FORWARD 1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -I FORWARD 1 -i "$TAP_DEV" -o "$HOST_IFACE" -j ACCEPT

API_SOCKET="/tmp/firecracker.socket"
LOGFILE="./firecracker.log"

# Create log file
touch $LOGFILE

# Set log file
sudo curl -X PUT --unix-socket "${API_SOCKET}" \
    --data "{
        \"log_path\": \"${LOGFILE}\",
        \"level\": \"Debug\",
        \"show_level\": true,
        \"show_log_origin\": true
    }" \
    "http://localhost/logger"

KERNEL="./$(ls vmlinux* | tail -1)"
KERNEL_BOOT_ARGS="console=ttyS0 reboot=k panic=1 pci=off"

ARCH=$(uname -m)

if [ ${ARCH} = "aarch64" ]; then
    KERNEL_BOOT_ARGS="keep_bootcon ${KERNEL_BOOT_ARGS}"
fi

# Set boot source
sudo curl -X PUT --unix-socket "${API_SOCKET}" \
    --data "{
        \"kernel_image_path\": \"${KERNEL}\",
        \"boot_args\": \"${KERNEL_BOOT_ARGS}\"
    }" \
    "http://localhost/boot-source"

ROOTFS="./ubuntu-22.04.ext4"

# Set rootfs
sudo curl -X PUT --unix-socket "${API_SOCKET}" \
    --data "{
        \"drive_id\": \"rootfs\",
        \"path_on_host\": \"${ROOTFS}\",
        \"is_root_device\": true,
        \"is_read_only\": false
    }" \
    "http://localhost/drives/rootfs"

# The IP address of a guest is derived from its MAC address with
# `fcnet-setup.sh`, this has been pre-configured in the guest rootfs. It is
# important that `TAP_IP` and `FC_MAC` match this.
FC_MAC="06:00:AC:10:00:02"

# Set network interface
sudo curl -X PUT --unix-socket "${API_SOCKET}" \
    --data "{
        \"iface_id\": \"net1\",
        \"guest_mac\": \"$FC_MAC\",
        \"host_dev_name\": \"$TAP_DEV\"
    }" \
    "http://localhost/network-interfaces/net1"

# API requests are handled asynchronously, it is important the configuration is
# set, before `InstanceStart`.
sleep 0.015s

# Start microVM
sudo curl -X PUT --unix-socket "${API_SOCKET}" \
    --data "{
        \"action_type\": \"InstanceStart\"
    }" \
    "http://localhost/actions"
```

Going back to your first shell, you should now see guest VM's prompt.

We have to finish network setup **in the guest**.
```
ip addr add 172.16.0.2/24 dev wlan0
ip link set wlan0 up
ip route add default via 172.16.0.1 dev wlan0

# echo nameserver 8.8.8.8 > /etc/resolv.conf
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
API_SOCKET="/tmp/firecracker.socket"

# Remove API unix socket
sudo rm -f $API_SOCKET

# Run firecracker
sudo ./firecracker --api-sock "${API_SOCKET}"
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
    "vcpu_count": 4,
    "mem_size_mib": 2048,
    "smt": false,
    "track_dirty_pages": false
  },
  "balloon": null,
  "network-interfaces": [
    {
      "iface_id": "wlan0",
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
You can check other options on [here](https://github.com/firecracker-microvm/firecracker/blob/main/src/firecracker/swagger/firecracker.yaml).

Also, you have to do same thing to use network in the guest.

If you want to power-off the guest VM, run `reboot` in the guest terminal.

**Whenever you reboot guest VM, you should remove socket.**
```
rm -f /tmp/firecracker.socket
```

otherwise, you can see this error.
```
[anonymous-instance:fc_api:ERROR:src/api_server/src/lib.rs:174] Error creating the HTTP server: IO error: Address in use (os error 98)
```

You can use ssh to enter guest VM.
```
ssh -i ./ubuntu-22.04.id_rsa root@172.16.0.2
```

**Advanced:** If you are running multiple Firecracker MicroVMs in parallel, or have something else on your system using tap0 then you need to create a tap for each one, with a unique name.

**Advanced:** You also need to do the iptables set up for each new tap. If you have iptables rules you care about on your host, you may want to save those rules before starting.
```
sudo iptables-save > iptables.rules.old
```

For convenience, you can make run script.
```
./fc_run.sh
```

```
#!/bin/bash

sudo ip tuntap add tap0 mode tap
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i tap0 -o wlan0 -j ACCEPT

rm -f /tmp/firecracker.socket
./firecracker --api-sock /tmp/firecracker.socket --config-file vm_config.json
```

### References
- https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md
- https://github.com/firecracker-microvm/firecracker/blob/main/docs/network-setup.md
- https://dev.to/l1x/getting-started-with-firecracker-on-raspberry-pi-1pbc
- https://s8sg.medium.com/quick-start-with-firecracker-and-firectl-in-ubuntu-f58aeedae04b

### TODO
- [x] Whenever boot guest VM, I have to do network setup in the guest everytime. Is there any solution to do this only once?

**[Solution]**

First, create network configuration file.
```
cat << EOF | tee /etc/systemd/network/my-network-config.network > /dev/null
[Match]
Name=eth0

[Network]
Address=172.16.0.2/24
Gateway=172.16.0.1
EOF
```

Then, enable and restart the network service. (name of the service is a little bit weird...)
```
systemctl enable systemd-networkd.service
systemctl restart systemd-networkd.service
```
