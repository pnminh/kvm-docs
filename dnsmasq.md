# Use dnsmasq as part of Network Manager
As `Network Manager` is integrated heavily to Linux, it does support very well different networking scenarios, including adding different DNS servers when a VPN connection is established.   

For example, here is the log (full log is in `network-manager-openvpn.log`) from using `journalctl` when an Open VPN connection is established using `network-manager-openvpn` plugin
```bash
$ journalctl -u NetworkManager -f|grep -i vpn
...
May 04 18:09:47 mpham NetworkManager[1283380]: <info>  [1588633787.7589] vpn-connection[0x55ded3b2c170,f281b867-85e1-4979-8adf-ad9fac216a7c,"Raleigh (RDU2)",30:(tun0)]: Data:   Internal DNS: 10.11.5.19
May 04 18:09:47 mpham nm-openvpn[1291869]: GID set to nm-openvpn
May 04 18:09:47 mpham NetworkManager[1283380]: <info>  [1588633787.7589] vpn-connection[0x55ded3b2c170,f281b867-85e1-4979-8adf-ad9fac216a7c,"Raleigh (RDU2)",30:(tun0)]: Data:   Internal DNS: 10.5.30.160
May 04 18:09:47 mpham nm-openvpn[1291869]: UID set to nm-openvpn
May 04 18:09:47 mpham NetworkManager[1283380]: <info>  [1588633787.7589] vpn-connection[0x55ded3b2c170,f281b867-85e1-4979-8adf-ad9fac216a7c,"Raleigh (RDU2)",30:(tun0)]: Data:   DNS Domain: 'redhat.com'
...
```
We can also get the info from querying Network Manager connection command 
```bash
$ nmcli connection show <vpn_connection>
...
IP4.ROUTE[1]:                           dst = 10.0.0.0/8, nh = 10.10.112.1, mt = 50
IP4.ROUTE[2]:                           dst = 10.10.112.0/20, nh = 0.0.0.0, mt = 50
IP4.DNS[1]:                             10.11.5.19
IP4.DNS[2]:                             10.5.30.160
IP4.DOMAIN[1]:                          redhat.com
VPN.TYPE:                               openvpn
... 
```

By default `sytemd-resolved` is bound to `120.0.0.53:53` and works as a DNS stub for all DNS queries, meaning all queries will go to this IP. If `systemd-resolved` can find the cached DNS response, it will send it back to the requesting client. If not `systemd-resolved` makes a request to upstream DNS server, receives the response and sends it back to client as well as saves it into the cache for faster queries next time. 
We can use `dnsmasq` in place of `systemd-resolved` as a stub DNS. `dnsmasq` gives extra capabilities such as wildcard domain searching.
## Disable sytemd-resolved DNS stub   

This set up will let systemd-resolved to run but does not bind to port 53
```bash
$ sudo mkdir -p /etc/systemd/resolved.conf.d
$ cat << EOF | sudo tee /etc/systemd/resolved.conf.d/noresolved.conf
[Resolve]
DNSStubListener=no
EOF
```
Another option is to disable and stop `systemd-resolved` service
```bash
$ sudo systemctl disable systemd-resolved
$ sudo systemctl stop systemd-resolved
``` 
## Set up /etc/resolv.conf
The `/etc/resolv.conf` may be a symlink to `/run/systemd/resolve/resolv.conf` which is the dynamic list of all DNS servers. We can have Network Manager to override data to this file instead by simply delete the symlink. Network Manager can create the file if it does not exist and add its own list there.
```bash
$ rm -rf /etc/resolv.conf
```
## Set up dnsmasq
Add dns configuration to Network Manager service and restart it. The content from the file `/etc/resolv.conf` is also updated with the IP for dnsmasq service.
```bash
$ cat << EOF | sudo tee /etc/NetworkManager/conf.d/dns.conf
[main]
dns=dnsmasq
EOF
$ sudo systemctl restart NetworkManager
$ cat /etc/resolv.conf 
# Generated by NetworkManager
search Home
nameserver 127.0.1.1
```
We can also see that dnsmasq is started by Network Manager
```bash
$ ps aux|grep dnsmasq
nobody   1331411  0.0  0.0  17856  3608 ?        S    18:50   0:00 /usr/sbin/dnsmasq --no-resolv --keep-in-foreground --no-hosts --bind-interfaces --pid-file=/run/NetworkManager/dnsmasq.pid --listen-address=127.0.1.1 --cache-size=0 --clear-on-reload --conf-file=/dev/null --proxy-dnssec --enable-dbus=org.freedesktop.NetworkManager.dnsmasq --conf-dir=/etc/NetworkManager/dnsmasq.d
```
## Add dnsmasq configurations files
We can add configuration files to `/etc/NetworkManager/dnsmasq.d` to update `dnsmasq` behaviors. Here are a couple of examples.
### Add wildcard domains
```bash
$ cat /etc/NetworkManager/dnsmasq.d/crc.conf 
address=/apps-crc.testing/192.168.122.44
address=/crc.testing/192.168.122.44
``` 
`apps-crc.testing` and `crc.testing` and all of their subdomains will be resolved to `192.168.122.44`
### Use upstream dns servers
In this example google DNS is used
```bash
$ cat /etc/NetworkManager/dnsmasq.d/google.conf 
server=8.8.8.8
server=8.8.4.4
```
### Logging configurations
We can enable `dnsmasq` logging and specify the location of the log file
```bash
$ cat /etc/NetworkManager/dnsmasq.d/logging.conf 
log-queries
log-facility=/var/log/dnsmasq.log
```
### Query all upstream servers
We can have dnsmasq query all upstream servers concurrently and return the reply from the server with the fastest response.
```bash
$ cat /etc/NetworkManager/dnsmasq.d/all_servers.conf 
all-servers
```
### Enable caching
By default Network Manager's dnsmasq does not have caching enabled. We can see the cache option `--cache-size=0` with the `ps` command. 
```bash
$ ps aux|grep dnsmasq
nobody   1331411  0.0  0.0  17856  3608 ?        S    18:50   0:00 /usr/sbin/dnsmasq --no-resolv --keep-in-foreground --no-hosts --bind-interfaces --pid-file=/run/NetworkManager/dnsmasq.pid --listen-address=127.0.1.1 --cache-size=0 --clear-on-reload --conf-file=/dev/null --proxy-dnssec --enable-dbus=org.freedesktop.NetworkManager.dnsmasq --conf-dir=/etc/NetworkManager/dnsmasq.d
```
We can override this option by adding a configuration file with the flag `cache-size`
```bash
$ cat /etc/NetworkManager/dnsmasq.d/cache.conf 
cache-size=1000
```
# Standalone dnsmasq as local DNS server
```
Note*: This doesn't play well with Network Manager OpenVPN plugin as the DNS list is not updated as in the case of using Network Manager's dnsmasq
```
## Set up standalone dnsmasq as the default local DNS server
First, install `dnsmasq`
```bash
$ sudo apt install dnsmasq
```
Disable `systemd-resolved` which is the default dns server stub
```bash
$ sudo systemctl stop systemd-resolved
$ sudo systemctl disable systemd-resolved
```
Or if we want to keep `systemd-resolved` running but not binding to port `53`, adding a file at `/etc/systemd/resolved.conf.d` with the content
```bash
$ sudo mkdir -p /etc/systemd/resolved.conf.d
$ cat << EOF | sudo tee /etc/systemd/resolved.conf.d/noresolved.conf
[Resolve]
DNSStubListener=no
EOF
$ sudo systemctl restart systemd-resolved
```
We also need to disable overriding of `/etc/resolv.conf` by `NetworkManager` (and also avoid `NetworkManager` from spinning its own dnsmasq server- not confirmed but observed locally)
```bash
$ cat << EOF | sudo tee /etc/NetworkManager/conf.d/disableresolv.conf
[main]
dns=none
EOF
$ sudo systemctl restart NetworkManager
```
Start and enable `dnsmasq`
```bash
$ sudo systemctl enable dnsmasq
$ sudo systemctl start dnsmasq
```
## Set up /etc/resolv.conf for DNS lookup
To dynamically update `/etc/resolv.conf` (dns resolver), install `resolvconf` to dynamically compute the resolver list from info collected from `/etc/resolvconf/resolv.conf.d/{head,base,tail}` and from the system.
```bash
$ sudo apt install resolvconf
```
Update the `/etc/resolvconf/resolv.conf.d/tail` file with additional resolvers, since `resolvconf` can see that `dnsmasq` is running on `127.0.0.1:53` and add that automatically to dns resolver list. The resolvers from the `tail` file will be appended after the local resolver(`127.0.0.1`).
```bash
$ cat << EOF | sudo tee /etc/resolvconf/resolv.conf.d/tail
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```
Link `/etc/resolv.conf` to `/run/resolvconf/resolv.conf` which is the the computed `resolv.conf` by the `resolvconf` service
```bash
$ sudo ln -s /run/resolvconf/resolv.conf /etc/resolv.conf
```
Update `resolvconf` with the new data
```bash
$ sudo resolvconf -u
$ cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
# 127.0.0.53 is the systemd-resolved stub resolver.
# run "systemd-resolve --status" to see details about the actual nameservers.
nameserver 127.0.0.1
nameserver 8.8.8.8
nameserver 8.8.4.4
```
Enable and start `resolvconf` so it can watch changes to the system's dns services
```bash
$ sudo systemctl enable resolvconf
$ sudo systemctl start resolvconf
```
## Set up wildcard domains
If we want to add a whole subdmain dns resolution (wildcard domain), create a new config file in `/etc/dnsmasq.d/` or update `/etc/dnsmasq.conf`
```bash
$ cat << EOF | sudo tee /etc/dnsmasq.d/crc.conf
address=/apps-crc.testing/192.168.122.44
address=/crc.testing/192.168.122.44
EOF
```
The `crc.conf` created above will resolve all domains ending with `apps-crc.testing` and `crc.testing` with the IP `192.168.122.44`. Restart `dnsmasq` and test the dns resolution
```bash
$ sudo systemctl restart dnsmasq
$ dig +noall +answer crc.testing
crc.testing.            0       IN      A       192.168.122.44
$ dig +noall +answer api.crc.testing
api.crc.testing.        0       IN      A       192.168.122.44
$ dig +noall +answer apps-crc.testing
apps-crc.testing.       0       IN      A       192.168.122.44
$ dig +noall +answer console.apps-crc.testing
console.apps-crc.testing. 0     IN      A       192.168.122.44
```
## FAQs
1. `address` vs `server` in config file   
   
   For example
    ```bash
    address=/example.com/192.168.1.10
    server=/example.com/192.168.1.10
    ```
   - `address`: example.com will be resolved to 192.168.1.10. In this case dnsmasq is used as a dns server
   - `server`: any dns lookup with `example.com` as parent domain will be sent to the dns server `192.168.1.10`. The dns server can return the same or different IP for `example.com`, e.g. `192.168.1.11`.
2. Set up logging in config file   
   
   For example,
   ```bash
    log-queries
    log-facility=/var/log/dnsmasq.log
   ```
   will add logs to `/var/log/dnsmasq.log`
# ToDo: dnsmasq as DHCP server
## References
- [How to avoid conflicts between dnsmasq and systemd-resolved?](https://unix.stackexchange.com/questions/304050/how-to-avoid-conflicts-between-dnsmasq-and-systemd-resolved)
- [How to disable systemd-resolved and resolve DNS with dnsmasq?](https://askubuntu.com/questions/898605/how-to-disable-systemd-resolved-and-resolve-dns-with-dnsmasq)
- [How do I set my DNS when resolv.conf is being overwritten?](https://unix.stackexchange.com/questions/128220/how-do-i-set-my-dns-when-resolv-conf-is-being-overwritten)
- [Configure DNS Wildcard with Dnsmasq Service](https://qiita.com/bmj0114/items/9c24d863bcab1a634503)
- [Wildcard subdomains with dnsmasq](https://stackoverflow.com/questions/22313142/wildcard-subdomains-with-dnsmasq)
- [Option all-servers for multiple DNS servers to enable minimizing the latency](https://discourse.pi-hole.net/t/option-all-servers-for-multiple-dns-servers-to-enable-minimizing-the-latency/5930)
- [Using the NetworkManager’s DNSMasq plugin](https://fedoramagazine.org/using-the-networkmanagers-dnsmasq-plugin/)
- [Using dnsmasq with NetworkManager](https://superuser.com/questions/681993/using-dnsmasq-with-networkmanager)