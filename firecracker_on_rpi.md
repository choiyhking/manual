# Guide to Getting Started with Firecracker on Raspberry Pi

[Getting Started with Firecracker](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md)

- Information

Model: **Raspberry Pi 4 Model B(8GB)**

OS: **Ubuntu Desktop 22.04.3 LTS(64-bit)**

Kernel version: **5.15.0-1034-raspi**

- ARM에서 KVM 지원 여부 확인하기

```bash
1. 커널 메시지를 통해 kvm 사용 확인

sudo dmesg | grep -i kvm

결과 > 
[    0.324955] kvm [1]: IPA Size Limit: 44 bits
[    0.326588] kvm [1]: vgic interrupt IRQ9
[    0.326858] kvm [1]: Hyp mode initialized successfully

2. 패키지를 통해 kvm 지원 확인

sudo apt install -y cpu-checker
kvm-ok

결과 >
INFO: /dev/kvm exists
KVM acceleration can be used
```

- KVM 패키지 설치

```bash
sudo apt install -y qemu-kvm libvirt-daemon bridge-utils libvirt-clients virtinst virt-manager
```

- KVM 설치 확인

```bash
# libvirtd 설치 확인
sudo systemctl is-active libvirtd

결과 > active

# 사용자 그룹 추가
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# libvirtd 서비스 확인
sudo systemctl enable libvirtd --now
sudo systemctl status libvirtd.service
```

- Docker 설치

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
docker --version
```

- Docker Group에 사용자 추가하기(optional)

```bash
sudo usermod -aG docker $USER
sudo service docker restart

그래도 완전 적용이 되지 않는다면 reboot
```

- Getting a rootfs and Guest Kernel Image

```bash
# Download a linux kernel binary
ARCH="$(uname -m)"
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.6/${ARCH}/vmlinux-5.10.198

# Download a rootfs
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.6/${ARCH}/ubuntu-22.04.ext4

# Download the ssh key for the rootfs
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.6/${ARCH}/ubuntu-22.04.id_rsa

# Set user read permission on the ssh key
chmod 400 ./ubuntu-22.04.id_rsa
```

- Getting a Firecracker Binary

```bash
ARCH="$(uname -m)"
release_url="https://github.com/firecracker-microvm/firecracker/releases"
latest=$(basename $(curl -fsSLI -o /dev/null -w  %{url_effective} ${release_url}/latest))
curl -L ${release_url}/download/${latest}/firecracker-${latest}-${ARCH}.tgz | tar -xz

# Rename the binary to "firecracker"
release-v1.5.0-aarch64
mv release-${latest}-$(uname -m)/firecracker-${latest}-${ARCH} firecracker
```

```bash
# Clone the firecracker repository
ARCH="$(uname -m)"
git clone https://github.com/firecracker-microvm/firecracker firecracker_src

# Build firecracker
## It is possible to build for gnu, by passing the arguments '-l gnu'.
## This will produce the firecracker and jailer binaries under
# `./firecracker/build/cargo_target/${toolchain}/debug`.
#
sudo ./firecracker_src/tools/devtool build
# 시간 오래 걸림!!

# Rename the binary to "firecracker"
sudo cp ./firecracker_src/build/cargo_target/${ARCH}-unknown-linux-musl/debug/firecracker firecracker
```

```bash
truncate -s 8G ubuntu-22.04.ext4
resize2fs ubuntu-22.04.ext4
```

```bash
vim vm_config.json

{
  "boot-source": {
    "kernel_image_path": "./vmlinux-5.10.198",
    "boot_args": "keep_bootcon console=ttyS0 reboot=k panic=1 pci=off",
    "initrd_path": null
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "partuuid": null,
      "is_root_device": true,
      "cache_type": "Unsafe",
      "is_read_only": false,
      "path_on_host": "./ubuntu-22.04.ext4",
      "io_engine": "Sync",
      "rate_limiter": null,
      "socket": null
    }
  ],
  "machine-config": {
    "vcpu_count": 4,
    "mem_size_mib": 4096,
    "smt": false,
    "track_dirty_pages": false
  },
  "cpu-config": null,
  "balloon": null,
  "network-interfaces": [
    {
      "iface_id": "eth0",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "tap0"
    }
  ],
  "vsock": null,
  "logger": {
    "log_path": "./firecracker.log",
    "level": "Debug",
    "show_level": true,
    "show_log_origin": true
  },
  "metrics": null,
  "mmds-config": null,
  "entropy": null,
  "InstanceActionInfo": {
    "action_type": "InstanceStart"
  }
}
```

```bash
API_SOCKET="/tmp/firecracker.socket"
sudo rm -f $API_SOCKET

# Setup network interface
TAP_DEV="tap0"
TAP_IP="172.16.0.1"
MASK_SHORT="/24"

sudo ip link del "$TAP_DEV" 2> /dev/null || true
sudo ip tuntap add dev "$TAP_DEV" mode tap
sudo ip addr add "${TAP_IP}${MASK_SHORT}" dev "$TAP_DEV"
sudo ip link set dev "$TAP_DEV" up

# Enable ip forwarding
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
# 아래의 오류 발생하면 root 계정으로 바꿔서 진행
`Set: 1: cannot create /proc/sys/net/ipv4/ip_forward#: Directory nonexistent`

# Set up microVM internet access
sudo iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE || true
sudo iptables -D FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT \ || true
sudo iptables -D FORWARD -i tap0 -o eth0 -j ACCEPT || true
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -I FORWARD 1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -I FORWARD 1 -i tap0 -o eth0 -j ACCEPT

LOGFILE="./firecracker.log"
touch $LOGFILE

./firecracker --api-sock /tmp/firecracker.socket --config-file vm_config.json

# [다른 터미널에서]
ssh -i ./ubuntu-22.04.id_rsa root@172.16.0.2

# [접속한 컨테이너 내부에서 계속 진행]
ip addr add 172.16.0.2/24 dev eth0
ip link set eth0 up
ip route add default via 172.16.0.1 dev eth0

# dns 주소 넣어주기
echo nameserver 155.230.10.2 > /etc/resolv.conf

# 명령어 확인
ping google.com -> 동작

apt update -> 에러

Reading package lists... Error!
E: flAbsPath on /var/lib/dpkg/status failed - realpath (2: No such file or directory)
E: Could not open file  - open (2: No such file or directory)
E: Problem opening
E: The package lists or status file could not be parsed or opened.

# 해결
mkdir /var/lib/dpkg
touch /var/lib/dpkg/status

apt install vim -> 정상 동작

# kernel version 다른 것 확인
`root@ubuntu-fc-uvm:~# uname -r`
5.10.198

`yunha@yunha-RaspPi4:~$ uname -r`
5.15.0-1034-raspi
```

![image](https://github.com/choiyhking/kata-and-fc-setup/assets/61413047/02983a91-6d32-43aa-b94b-cf1425e306c2)


- Other errors

USB에 microSD 사용 금지
