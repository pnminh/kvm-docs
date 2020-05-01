# KVM Documentation
## Install kvm
Fedora
```bash
$ sudo dnf -y install bridge-utils libvirt virt-install qemu-kvm virt-manager
```
or install group `Virtualization`
```bash
$ sudo dnf group install Virtualization
```
Ubuntu
```bash
$ sudo apt-get install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```
Add current user to `libvirt` and `kvm` groups
```bash
$ sudo adduser `id -un` libvirtd
$ sudo adduser `id -un` libvirt
$ sudo adduser `id -un` kvm
```
May need to log out for the group add to take effect. We can test by running 
```bash
$ groups
```

## Run Fedora/CentOS cloud image
Fedora/CentOS cloud is a minumum base image. This post is about installing and configuring [Fedora cloud](https://alt.fedoraproject.org/cloud/).
### Install Fedora Cloud VM
Download KVM image
```bash
$ wget https://download.fedoraproject.org/pub/fedora/linux/releases/32/Cloud/x86_64/images/Fedora-Cloud-Base-32-1.6.x86_64.qcow2
```

We can use this image directly. However we may need to resize the image as the original size may not a accommodate future needs.
```bash
# check size
$ qemu-img info Fedora-Cloud-Base-32-1.6.x86_64.qcow2 
image: Fedora-Cloud-Base-32-1.6.x86_64.qcow2
file format: qcow2
virtual size: 4 GiB (4294967296 bytes)
disk size: 289 MiB
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16

# resize image
$ qemu-img resize Fedora-Cloud-Base-32-1.6.x86_64.qcow2 50G
```
The cloud image uses `cloud-init` to set things up at boot time such hostname, usernames, passwords/ssh keys, etc. It's similar to ignition file for CoreOS platform. We will set up 2 files for `cloud-init`: `meta-data` and `user-data`

```bash
$ cat > meta-data << EOF
instance-id: mpham
local-hostname: mpham
EOF
$ cat > user-data << EOF
#cloud-config
# Set the default user
system_info:
  default_user:
    name: mpham
 
# Set password
chpasswd:
  list: |
     mpham:123456
  expire: False
 
# Other settings
resize_rootfs: True
ssh_pwauth: True
timezone: America/Chicago
 
# Add any ssh public keys
ssh_authorized_keys:
 - ssh-rsa AA..BC mpham@redhat.com
 
bootcmd:
 - [ sh, -c, echo "=========bootcmd=========" ]
  
runcmd:
 - [ sh, -c, echo "=========runcmd=========" ]
  
# For pexpect to know when to log in and begin tests
final_message: "SYSTEM READY TO LOG IN"
EOF
```
We can package these 2 files into ISO file and provides it as a cd input for `cloud-init`
```bash
$ genisoimage -output fedora_cloud32.iso -volid cidata -joliet -rock user-data meta-data
```
We then can create the VM from the 2 files: `Fedora-Cloud-Base-32-1.6.x86_64.qcow2` and `fedora_cloud32.iso`
```bash
$ virt-install --connect qemu:///system \
-n fedora_cloud -r 8192 --vcpus=4 -w network=default \
--import --disk path=./Fedora-Cloud-Base-32-1.6.x86_64.qcow2 \
--disk path=fedora_cloud32.iso,device=cdrom
```
### Set static IP for the new VM
Get the list of current networks used by KVM
```bash
$ virsh net-list
 Name              State    Autostart   Persistent
----------------------------------------------------
 crc               active   yes         yes
 default           active   yes         yes
 docker-machines   active   yes         yes
```
Usually the new VM by default will use the `default` network. Back up the default network info in case the configuration goes wrong
```bash
$ virsh net-dumpxml default > default_network_bk.xml
```
Get the MAC address for the VM
```bash
$ virsh dumpxml <VM_name> | grep -i '<mac'
      <mac address='52:54:00:fc:a5:72'/>
```
Then edit the `default` network, append `<host mac='52:54:00:fc:a5:72' name='vm_name' ip='192.168.122.33'/>` at the same level with `<range start='start_ip' end='end_ip'/>`,
with `mac`: the mac address retrieved above, the `name` can be anything but needs to be unique since all VMs shares this same network, so it's smart to use the VM name, and `ip` is the fixed IP to be used
```bash
$ virsh net-edit default
<network>
  <name>default</name>
  <uuid>4699892c-c4d8-4331-a380-02881cb07cba</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:49:00:15'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
        <range start='192.168.122.2' end='192.168.122.254'/>
        <host mac='52:54:00:fc:a5:72' name='fedora_cloud' ip='192.168.122.33'/>
    </dhcp>
  </ip>
</network>
```
Save the file and restart the DHCP service
```bash
$ virsh net-destroy default
$ virsh net-start default
```
 
We then need to shut down the VM

```bash
$ virsh shutdown <vm_name>
$ virsh start <vm_name>
$ ssh mpham@192.168.122.33
```
Since we already inserted the ssh public key into the `user-data` file, we should expect to log in to the VM without password prompt.
 
## Resize VM disk
First the VM needs to be shut down
```bash
$ virsh shutdown fedora_cloud
```
Get the path for the backing storage
```bash
$ virsh domblklist fedora_cloud 
 Target   Source
---------------------------------------------------------------------------------
 hda      /home/minhpn/kvm/fedora_cloud32/Fedora-Cloud-Base-32-1.6.x86_64.qcow2
 hdb      /home/minhpn/kvm/fedora_cloud32/fedora_cloud32.iso
```
Resize the disk
```bash
$ sudo qemu-img resize /home/minhpn/kvm/fedora_cloud32/Fedora-Cloud-Base-32-1.6.x86_64.qcow2 +10G
```
Make sure there is `no snapshot` taken for the VM, otherwise we cannot resize the disk.
## KVM vs QEMU vs Libvirt
- QEMU: Hypervisor/emulator, working by a special 'recompiler' that transforms binary code written for a given processor into another one (e.g. ARM in an x86 PC). QEMU also emulates peripheral components.
- KVM: Linux kernel module, accelerating agent that optimizes QEMU hypervisor on different hardware achitectures. Working together with QEMU with hardware virtualiztion enabled, KVM takes care of CPU and memory, and QUEMU does that for peripherals.
- Libvirt: Virtualization library that provides API library, a `libvirtd` deamon, and `virsh` CLI client. 
## References:
- [Fedora-Installing Virtualization package groups](https://docs.fedoraproject.org/en-US/Fedora/22/html/Virtualization_Getting_Started_Guide/ch06s02.html)
- [Ubuntu KVM/Installation](https://help.ubuntu.com/community/KVM/Installation)
- [Booting Fedora cloud images with KVM](https://blog.christophersmart.com/2016/06/17/booting-fedora-24-cloud-image-with-kvm/)
- [Using Cloud Images in KVM](https://www.theurbanpenguin.com/using-cloud-images-in-kvm/)
- [KVM libvirt assign static guest IP addresses using DHCP on the virtual machine](https://www.cyberciti.biz/faq/linux-kvm-libvirt-dnsmasq-dhcp-static-ip-address-configuration-for-guest-os/)
- [How To extend/increase KVM Virtual Machine (VM) disk size](https://computingforgeeks.com/how-to-extend-increase-kvm-virtual-machine-disk-size/)
- [KVM vs QEMU vs Libvirt](https://www.thegeekyway.com/kvm-vs-qemu-vs-libvirt/)
- [Difference between KVM and QEMU](https://serverfault.com/questions/208693/difference-between-kvm-and-qemu)
- [What is the difference and relationship between kvm, virt-manager, qemu and libvirt?](https://superuser.com/questions/1490188/what-is-the-difference-and-relationship-between-kvm-virt-manager-qemu-and-libv)