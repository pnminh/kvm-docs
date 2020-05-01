# Set up CodeReady Containers on KVM VM
## Host Pass-through (Nested) Virtualization
Check if nested virtualization is supported (should be for modern hardware)
```bash
$ cat /sys/module/kvm_intel/parameters/nested
```
Enable `host-model` for the VM's CPU configurations. 
Go to `Virtual Machine Manager` > Double-click on the VM > Choose `View` > `Details` > `CPU` > Select `Copy host CPU configuration`.  

```
Note: This is the default option for the new VM
```
On the guest client, make sure virtualization packages are installed. Here is an example of Fedora VM
```bash
$ sudo dnf group install virtualization
```
Verify that the virtual machine has virtualization correctly set up:
```bash
$ virt-host-validate
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
```

## Install and start CodeReady Containers with crc
## Set up host's dnsmasq to resolve guest's OCP wildcard routes
## Set up Port-forwarding for nested VMs
## References
