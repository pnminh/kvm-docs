# Set up CodeReady Containers (crc) on KVM VM
This documentation is for installing crc on Fedora Cloud VM. 
## Notations
- `[user@host]$`: command to be run on host
- `[user@guest]$`: command to be run on guest VM
## Topology
- Host IP: 192.168.1.10
- Fedora Cloud IP: 192.168.122.33
- crc nested VM: 192.168.130.11
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
## Set up Port-forwarding for nested VMs

By default, only the host can see the guest VM network. In the case of crc installed on Fedora Cloud guest VM, there is a new nested VM created by crc to run Openshift that can only be seen by Fedora Cloud.   

To have the host see Openshift network, we need to do Port Forwarding using `iptables` inside Fedora Cloud VM to forward traffic to Openshift. Several common ports include 80 and 443 for application routes and 6443 for API route.  

Given the Openshift VM uses `crc` network (this network is configured when we ran `crc setup` above) with the VM IP `192.168.130.11`.  

On the host/guest (both can see the `crc` network), we can get the info about the crc network with the fixed IP `192.168.130.11` set for the Openshift VM:
```bash
[user@host]$ virsh net-dumpxml crc | tee crc_network.xml
<network>
  <name>crc</name>
  <uuid>49eee855-d342-46c3-9ed3-b8d1758814cd</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='crc' stp='on' delay='0'/>
  <mac address='52:54:00:fd:be:d0'/>
  <ip family='ipv4' address='192.168.130.1' prefix='24'>
    <dhcp>
      <host mac='52:fd:fc:07:21:82' ip='192.168.130.11'/>
    </dhcp>
  </ip>
</network>
```
On Fedora Cloud VM, update `iptables` to forward traffic to the Openshift VM 
```bash
[user@guest]$ sudo iptables -I FORWARD -o crc -d 192.168.130.11 -j ACCEPT
# forward port 80
[user@guest]$ sudo iptables -t nat -I PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.130.11:80
# forward port 443
[user@guest]$ sudo iptables -t nat -I PREROUTING -p tcp --dport 443 -j DNAT --to 192.168.130.11:443
# forward port 6443
[user@guest]$ sudo iptables -t nat -I PREROUTING -p tcp --dport 6443 -j DNAT --to 192.168.130.11:6443
```
To persist the `iptables` rules, we would need to install iptables-services. Also `iptables` may conflict with `firewalld` so better to disable `firewalld` before running iptables
```bash
[user@guest]$ sudo systemctl disable firewalld
[user@guest]$ sudo systemctl disable firewalld
[user@guest]$ sudo systemctl enable iptables
[user@guest]$ sudo service iptables save
```

## Set up host's dnsmasq to resolve guest's OCP wildcard routes

There are 2 routes that OCP exposes: the api route (`api.crc.testing`) that uses the parent domain `.crc.testing`  and the application wildcard route (`*.apps-crc.testing`) which includes the console URL.

From the host, edit the file `/etc/dnsmasq.conf` and add the lines with the IP set to the Fedora Cloud VM IP, which in place with forward the traffic to crc VM:
```bash
[user@host]$ sudo vi /et/dnsmasq.conf
# Add domains which you want to force to an IP address here.
# The example below send any host in double-click.net to a local
# web-server.
#address=/double-click.net/127.0.0.1
address=/.apps-crc.testing/192.168.122.33
address=/.crc.testing/192.168.122.33
```
The above configurations can also be observed with querying the crc dnsmasq config file on Fedora Cloud guest VM, except the IP is of the crc VM:
```bash
[user@guest]$ cat /etc/NetworkManager/dnsmasq.d/crc.conf
server=/apps-crc.testing/192.168.130.11
server=/crc.testing/192.168.130.11
```
Restart dnsmasq service and run dig to check for dns resolution
```bash
[user@host]$ sudo systemctl restart dnsmasq
[user@host]$ dig +noall +answer api.crc.testing
api.crc.testing.        0       IN      A       192.168.122.33
[user@host]$ dig  +noall +answer console.apps-crc.testing
console.apps-crc.testing. 0     IN      A       192.168.122.33
```
## SSH to crc VM
Look into `crc.log` file to get the ssh key that crc installation used to ssh to crc VM. We can see there were 2 keys mentioned from the logs. I tried both but only the `~/.crc/machines/crc/id_rsa` worked for me.
```bash
[user@guest]$ cat ~/.crc/crc.log |grep ssh
&{[-F /dev/null -o ConnectionAttempts=3 -o ConnectTimeout=10 -o ControlMaster=no -o ControlPath=none -o LogLevel=quiet -o PasswordAuthentication=no -o ServerA
liveInterval=60 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null core@192.168.130.11 -o IdentitiesOnly=yes -i /home/mpham/.crc/cache/crc_libvirt_4.
3.10/id_rsa_crc -p 22] /usr/bin/ssh <nil>}                                                                                                                    
&{[-F /dev/null -o ConnectionAttempts=3 -o ConnectTimeout=10 -o ControlMaster=no -o ControlPath=none -o LogLevel=quiet -o PasswordAuthentication=no -o ServerA
liveInterval=60 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null core@192.168.130.11 -o IdentitiesOnly=yes -i /home/mpham/.crc/machines/crc/id_rsa 
-p 22] /usr/bin/ssh <nil>}
```
We then can also log in to crc VM using user`core`
```bash
[user@guest]$ ssh -i ~/.crc/machines/crc/id_rsa core@$(crc ip)
Red Hat Enterprise Linux CoreOS 43.81.202003310153.0
  Part of OpenShift 4.3, RHCOS is a Kubernetes native operating system
  managed by the Machine Config Operator (`clusteroperator/machine-config`).

WARNING: Direct SSH access to machines is not recommended; instead,
make configuration changes via `machineconfig` objects:
  https://docs.openshift.com/container-platform/4.3/architecture/architecture-rhcos.html

---
Last login: Wed May  6 02:35:02 2020 from 192.168.130.1
```
## References
- [CodeReady Containers wiki](https://code-ready.github.io/crc/)
- [KVM/libvirt: Forward Ports to guests with Iptables](https://aboullaite.me/kvm-qemo-forward-ports-with-iptables/)
- [Centos 7 save iptables settings](https://serverfault.com/questions/626521/centos-7-save-iptables-settings)
- [Configure DNS Wildcard with Dnsmasq Service](https://qiita.com/bmj0114/items/9c24d863bcab1a634503)
- [Wildcard subdomains with dnsmasq](https://stackoverflow.com/questions/22313142/wildcard-subdomains-with-dnsmasq)
