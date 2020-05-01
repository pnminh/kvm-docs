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
sudo apt-get install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
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
qemu-img resize Fedora-Cloud-Base-32-1.6.x86_64.qcow2 50G
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
$ virt-install --connect qemu:///system          -n fedora_cloud          -r 8192 --vcpus=4  -w network=default          --import          --disk path=./Fedora-Cloud-Base-32-1.6.x86_64.qcow2          --disk path=fedora_cloud32.iso,device=cdrom
```
### Set static IP for the new VM

References:
- [Booting Fedora cloud images with KVM](https://blog.christophersmart.com/2016/06/17/booting-fedora-24-cloud-image-with-kvm/)
- [Using Cloud Images in KVM](https://www.theurbanpenguin.com/using-cloud-images-in-kvm/)