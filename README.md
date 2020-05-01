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