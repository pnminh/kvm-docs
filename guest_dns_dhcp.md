# KVM: set up guest as DNS, DHCP, TFTP servers
By default `libvirt` daemon uses `dnsmasq` on host machine to provide DNS and DHCP services to guests. One example is the `default`network.   
We can set up a dedicated guest VM to serve these services instead. We can then extend this guest VM to serve other services like PXE, FTP, etc.   
In this doc, we will use dnsmasq to provide DNS, DHCP, and TFTP traffic.
## Network diagram
![Network Diagram](media/dns_dhcp_pxe.png "Network Diagram")
## Prerequisites
- Make sure VM1 with name `dhcp-server` was already created. Read [README.md](README.md) for information about running a Fedora Cloud machine.
## Set up libvirt network
libvirt already provides the `default` network
```bash
$ virsh net-list
 Name              State    Autostart   Persistent
----------------------------------------------------
 default           active   yes         yes
```

```bash
$ virsh net-dumpxml default 
<network>
  <name>default</name>
  <uuid>4699892c-c4d8-4331-a380-02881cb07cba</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:49:00:15'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
      <host mac='52:54:00:e1:b6:90' name='dhcp-server' ip='192.168.122.10'/>
    </dhcp>
  </ip>
</network>
```
Here we updated the network settings to provide static IP to a MAC address on our VM1 (`dhcp-server`) machine using `virsh net-edit default`.
```bash
$ virsh domiflist dhcp-server 
 Interface   Type      Source    Model    MAC
-------------------------------------------------------------
 -           network   default   virtio   52:54:00:e1:b6:90
 $ virsh net-edit default
```
Create a new isolated network. Note that we do not enable default DHCP by libvirt
```bash
$ cat <<EOF > isolated-dhcp-network.xml 
<network>
  <name>isolated-dhcp</name>
  <uuid>34c07436-c91c-451a-9fed-06417ddd1adc</uuid>
  <bridge name="virbr5" stp="on" delay="0"/>
  <mac address="52:54:00:90:25:bb"/>
  <ip address="10.120.121.1" netmask="255.255.255.0">
  </ip>
</network>
EOF
$ virsh net-create --file isolated-dhcp-network.xml 
Network isolated-dhcp created from isolated-dhcp-network.xml
```
Create a new network interface on `dhcp-server` VM and uses `isolated-dhcp` network
```bash
$ virsh attach-interface --domain dhcp-server --type network \
--source isolated-dhcp --model virtio \
--mac 52:54:00:4b:73:5f --config
$ virsh domiflist dhcp-server 
 Interface   Type      Source          Model    MAC
-------------------------------------------------------------------
 -           network   default         virtio   52:54:00:e1:b6:90
 -           network   isolated-dhcp   virtio   52:54:00:4b:73:5f
```
We are ready to set up `dnsmasq`. Let's start `dhcp-server` machine
```bash
$ virsh start dhcp-server 
Domain dhcp-server started
```
## Set up dnsmasq as DNS, DHCP, TFTP, and Gateway server
## Set up TFTP

## Set up NAT gateway for internet access

## Set up TFTP client

## References
- [VirtualNetworking](https://wiki.libvirt.org/page/VirtualNetworking)
- [iptables forwarding between two interface
](https://serverfault.com/questions/431593/iptables-forwarding-between-two-interface)
- [Setting up a ‘PXE Network Boot Server’ for Multiple Linux Distribution Installations in RHEL/CentOS 7](https://www.tecmint.com/install-pxe-network-boot-server-in-centos-7/)
- [Using network namespaces and a virtual switch to isolate servers](https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/)
- [Network Namespace , connect to host and with a VM](https://github.com/sjha3/Linux-Networking/wiki/5.-Network-Namespace-,-connect-to-host-and-with-a-VM)
- [Using network namespaces with veth to NAT guests with overlapping IPs](https://blog.christophersmart.com/2020/03/15/using-network-namespaces-with-veth-to-nat-guests-with-overlapping-ips/)