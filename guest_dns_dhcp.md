# KVM: set up guest as DNS, DHCP, TFTP servers
By default `libvirt` daemon uses `dnsmasq` on host machine to provide DNS and DHCP services to guests. One example is the `default`network.   
We can set up a dedicated guest VM to serve these services instead. We can then extend this guest VM to serve other services like PXE, FTP, etc.   
In this doc, we will use dnsmasq to provide DNS, DHCP, and TFTP traffic.
## Network diagram
![Network Diagram](media/dns_dhcp_pxe.png "Network Diagram")

## References
- [VirtualNetworking](https://wiki.libvirt.org/page/VirtualNetworking)
- [iptables forwarding between two interface
](https://serverfault.com/questions/431593/iptables-forwarding-between-two-interface)
- [Setting up a ‘PXE Network Boot Server’ for Multiple Linux Distribution Installations in RHEL/CentOS 7](https://www.tecmint.com/install-pxe-network-boot-server-in-centos-7/)
- [Using network namespaces and a virtual switch to isolate servers](https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/)
- [Network Namespace , connect to host and with a VM](https://github.com/sjha3/Linux-Networking/wiki/5.-Network-Namespace-,-connect-to-host-and-with-a-VM)
- [Using network namespaces with veth to NAT guests with overlapping IPs](https://blog.christophersmart.com/2020/03/15/using-network-namespaces-with-veth-to-nat-guests-with-overlapping-ips/)