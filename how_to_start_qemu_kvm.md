# How to start QEMU/KVM virtual machine

### Host specification
- Raspberry Pi 5
- Raspberry Pi OS (Debian GNU/Linux 12 (bookworm))
- 6.6.31+rpt-rpi-2712

### How-to
Check KVM module is enabled.
```
sudo dmesg | grep -i kvm

# Result >
# [    0.051877] kvm [1]: IPA Size Limit: 40 bits
# [    0.051900] kvm [1]: GICV region size/alignment is unsafe, using trapping (reduced performance)
# [    0.051927] kvm [1]: vgic interrupt IRQ9
# [    0.051940] kvm [1]: VHE mode initialized successfully
```

or run this command.
```
sudo apt install -y cpu-checker
kvm-ok

# Result >
# INFO: /dev/kvm exists
# KVM acceleration can be used
```

Install packages.
```
sudo apt install -y qemu-kvm libvirt-daemon libvirt-clients bridge-utils virtinst virt-manager
```
![image](https://github.com/user-attachments/assets/78972640-d7f5-44b3-ad51-53ebcf5fbf8f)

Check the libvirtd status.
```
sudo systemctl enable libvirtd --now
sudo systemctl is-active libvirtd

# Result > active
```

Add user to group.
```
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
sudo setfacl -m u:libvirt-qemu:rx $HOME
```

Check network list.
```
sudo virsh net-list

# Result >
# Name      State    Autostart   Persistent
# --------------------------------------------
# default   active   yes         yes

# if there's no default network, then run this
sudo virsh net-start default
sudo virsh net-autostart default
```

Create guest VM using virt GUI manager.
```
virt-manager
```

Create guest VM using commands.

First, you need OS image file.
```
wget https://cdimage.ubuntu.com/releases/noble/release/ubuntu-24.04.1-live-server-arm64.iso
```

then, create guest machine.
```
# virt-install --osinfo list | grep ubuntu

sudo virt-install --name=test-vm \
--vcpus=2 \
--memory=2048 \
--cdrom=ubuntu-24.04.1-live-server-arm64.iso \
--os-variant ubuntu22.04 \
--disk size=10
```
![image](https://github.com/user-attachments/assets/24e5d12d-a048-4efd-9720-43a7fd22acfb)


Then, you can see GRUB page.

**GRUB page**
(If you don't need customization, just keep going by defualt options)
1. Select `Install Ubuntu Server`

There's an error on the first line of output. (But it doesn't matter. Just a bug.)

`EFI stub: ERROR: FIRMWARE BUG: kernel image not aligned on 64k boundary`

Bug: https://bugs.launchpad.net/ubuntu/+source/grub2/+bug/1947046

2. Select `Switch to rich mode`
3. Language setting
4. Network connection
5. Congifure proxy
6. Configure Ubuntu archive mirror
7. Storage configuration

![스크린샷 2024-03-29 142852](https://github.com/yunhachoi/manual/assets/161846673/2ce82f24-54d6-4c22-921b-835d8b2e241b)

10. Confirm destructive action -> Select `continue`
11. Profile setup
12. SSH setup
13. Featured Server snaps
14. Installing system(it will take a while to complete)
15. Select `Reboot` -> error

`Failed unmounting /cdrom. Please remove the installation medium, then press enter`

-> press `ENTER`

-> no login prompt

-> `ctrl + c`

-> login

**Guest VM is created!!**

Check isolated environment btw host and guest.

**In the guest**
```
yunha@test-vm:~$ uname -a

# Result > 
# Linux debian 6.1.0-26-arm64 #1 SMP Debian 6.1.112-1 (2024-09-30) aarch64 GNU/Linux
```

**In the host**
```
pi@pi:~$ uname -a

# Result > 
# Linux raspberrypi 6.6.31+rpt-rpi-2712 #1 SMP PREEMPT Debian 1:6.6.31-1+rpt1 (2024-05-29) aarch64 GNU/Linux
```

We can check allocated guest VM's resources.

![스크린샷 2024-03-29 144746](https://github.com/yunhachoi/manual/assets/161846673/329456e7-8375-472d-b7fd-4824333f060d)

We can check automatically created disk images.
```
sudo ls /var/lib/libvirt/images/

# Result > test-vm.qcow2
```

### Commands
Logout.
```
exit 
```

Exit to host.
```
ctrl + ]
```

Show all the guest VMs.
```
virsh list --all

# Result >
# Id   Name      State
# -------------------------
# 10   test-vm   running
```

Connect the virtual serial console for the guest.
```
virsh console <VM>
```

Reboot.
```
virsh reboot <VM>
```

Power-off.
```
virsh shutdown <VM>
```

Start the guest machine.
```
virsh start <VM>
```

Immediately terminate the VM.

(This doesn't give the domain OS any chance to react, and it's the equivalent of ripping the power cord out on a physical machine.
It's better to use shutdown)
```
virsh destroy <VM>
```

Removing VM.
```
virsh undefine <VM>

# Result >
# error: Failed to undefine domain 'test-vm'
# error: Requested operation is not valid: cannot undefine domain with nvram

# --nvram remove nvram file
virsh undefine --nvram <VM>
```

## Trouble Shooting
- Error 1

`You will need to grant the 'libvirt-qemu' user search permissions for the following directories: ['/home/pi']`
```
sudo chmod +x /home/pi
```

- Error 2

`WARNING  Couldn't configure UEFI: Did not find any UEFI binary path for arch 'aarch64'`
```
sudo apt install qemu-efi-aarch64
```

then, execute virt-install with `--boot uefi` option.

## References
- Korean
  * https://linuxhint.com/kvm_virtualization_raspberry_pi4/
  * https://m.blog.naver.com/love_tolty/222650880951
  * https://yona.xslab.co.kr/엑세스랩/HOWTO-V-Raptor-SQ-nano/post/21
- English
  * https://linuxhint.com/kvm_virtualization_raspberry_pi4/
  * https://linux.die.net/man/1/virt-install
  * https://linux.die.net/man/1/virsh

## To-Do
- [ ] If I set disk size too small, I have to manually do storage configuration later.
But this makes booting error. For now, I don't know why.
