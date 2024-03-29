# How to create KVM guest machine

### Host spec
- Raspberry Pi 4 Model B(8GB)
- Ubuntu 22.04.4
- 5.15.0-1049-raspi

### How-to
Check KVM module is enabled.
```
sudo dmesg | grep -i kvm
# Result >
# [    0.328294] kvm [1]: IPA Size Limit: 44 bits
# [    0.329951] kvm [1]: vgic interrupt IRQ9
# [    0.330228] kvm [1]: Hyp mode initialized successfully

# or run this command 
sudo apt install -y cpu-checker
kvm-ok
# Result >
# INFO: /dev/kvm exists
# KVM acceleration can be used
```

Install packages.
```bash
sudo apt install -y qemu-kvm libvirt-daemon libvirt-clients bridge-utils virtinst virt-manager
```

```
# check the libvirtd status
sudo systemctl enable libvirtd --now
sudo systemctl is-active libvirtd
# Result > active

# Add user to group
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

```
# Check network
sudo virsh net-list
# Result >
# Name      State    Autostart   Persistent
# --------------------------------------------
# default   active   yes         yes
 
# if there's no default network, then run this
sudo virsh net-start default
```

Create guest Virtual Machine using virt GUI manager.
```bash
virt-manager
```

Create guest Virtual Machine using commands.
```
wget https://cdimage.ubuntu.com/releases/jammy/release/ubuntu-22.04.4-live-server-arm64.iso
# Result >
# -rw-rw-r--   1 libvirt-qemu kvm  1.9G  2월 20 09:52 ubuntu-22.04.4-live-server-arm64.iso

sudo virt-install --name=test-vm \
--vcpus=2 \
--memory=2048 \
--cdrom=./ubuntu-22.04.4-live-server-arm64.iso \
--disk size=10
```

### GRUB page
**If you don't need customization, just keep going by defualt options**
1. Install Ubuntu Server
2. Switch to rich mode
3. Language setting
4. Network connection
5. Congifure proxy
6. Configure Ubuntu archive mirror
7. Storage configuration
8. Confirm destructive action -> continue
9. Profile setup
10. SSH setup
11. Featured Server snaps
12. Installing system # it will take a while to complete
13. Reboot -> error 
`failed unmounting /cdrom, plz remove the installation medium, then press enter`
-> enter
-> no login prompt
-> ctrl+c -> login

```
# logout
exit 
# exit to host
ctrl + ]
```

```
yunha@test-vm:~$ uname -a
# Result > 
# Linux test-vm 5.15.0-101-generic #111-Ubuntu SMP Wed Mar 6 18:01:01 UTC 2024 aarch64 aarch64 aarch64 GNU/Linux

pi@pi:~$ uname -a
# Result > 
# Linux pi 5.15.0-1049-raspi #52-Ubuntu SMP PREEMPT Thu Mar 14 08:39:42 UTC 2024 aarch64 aarch64 aarch64 GNU/Linux
```

When booting, there's an error on the first line.(but it doesn't matter. just a bug.)

`EFI stub: ERROR: FIRMWARE BUG: kernel image not aligned on 64k boundary`

Bug: https://bugs.launchpad.net/ubuntu/+source/grub2/+bug/1947046

```
virsh list --all
# Result >
# Id   Name      State
# -------------------------
# 10   test-vm   running
```

```
sudo ls /var/lib/libvirt/images/
# Result > test-vm.qcow2
```

```
# Connect the virtual serial console for the guest
virsh console <VM>

virsh reboot <VM>

# Power-off
virsh shutdown <VM>

virsh start <VM>

# Immediately terminate the domain domain.
# This doesn't give the domain OS any chance to react, 
# and it's the equivalent of ripping the power cord out on a physical machine.
# It's better to use shutdown
virsh destroy <VM>

# Removing VM
virsh undefine <VM>
# Result >
# error: Failed to undefine domain 'test-vm'
# error: Requested operation is not valid: cannot undefine domain with nvram

virsh undefine --nvram <VM> # --nvram remove nvram file
```

## TODO

- [ ] if I set disk size too small, I have to config storage configuration later manually.
But this makes booting error. For now, I don't know why.
- [ ] Check virt-install --import option.
