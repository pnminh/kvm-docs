# Set up CodeReady Containers on KVM VM
## Notations
- `[user@host]$`: command to be run on host
- `[user@guest]$`: command to be run on guest VM
## Host Pass-through (Nested) Virtualization
Check if nested virtualization is supported (should be for modern hardware)
```bash
[user@host]$ cat /sys/module/kvm_intel/parameters/nested
```
Enable `host-model` for the VM's CPU configurations. 
Go to `Virtual Machine Manager` > Double-click on the VM > Choose `View` > `Details` > `CPU` > Select `Copy host CPU configuration`.  

```
Note: This is the default option for the new VM
```
On the guest client, make sure virtualization packages are installed. Here is an example of Fedora VM
```bash
[user@guest]$ sudo dnf group install virtualization
```
Verify that the virtual machine has virtualization correctly set up:
```bash
[user@guest]$ virt-host-validate
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
```

## Install and start CodeReady Containers (crc)
Download the latest version of [crc](https://cloud.redhat.com/openshift/install/crc/installer-provisioned)
```bash
[user@guest]$ wget https://mirror.openshift.com/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
[user@guest]$ tar -xvf crc-linux-amd64.tar.xz
[user@guest]$ sudo mv crc-linux-1.9.0-amd64/crc /usr/local/bin
[user@guest]$ crc setup
[user@guest]$ crc start
```
The `crc start` command requires a pull secret downloaded from [Red Hat OpenShift Cluster Manager](https://cloud.redhat.com/openshift/install/crc/installer-provisioned)
## Set up host's dnsmasq to resolve guest's OCP wildcard routes

There are 2 routes that OCP exposes: the api route (`api.crc.testing`) that uses the parent domain `.crc.testing`  and the application wildcard route (`*.apps-crc.testing`) which includes the console URL.

From the host, edit the file `/etc/dnsmasq.conf` and add the lines:
```bash
[user@host]$ sudo vi /et/dnsmasq.conf
# Add domains which you want to force to an IP address here.
# The example below send any host in double-click.net to a local
# web-server.
#address=/double-click.net/127.0.0.1
address=/.apps-crc.testing/192.168.122.33
address=/.crc.testing/192.168.122.33
```

Restart dnsmasq service and run dig to check for dns resolution
```bash
[user@host]$ sudo systemctl restart dnsmasq
[user@host]$ dig +noall +answer api.crc.testing
api.crc.testing.        0       IN      A       192.168.122.33
[user@host]$ dig  +noall +answer console.apps-crc.testing
console.apps-crc.testing. 0     IN      A       192.168.122.33
```
## Set up Port-forwarding for nested VMs

## References
- [CodeReady Containers wiki](https://code-ready.github.io/crc/)

