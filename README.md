# Notes on VLAN / DHCP / routing setup

## Packages

```
apt-get install vlan isc-dhcp-server iptables-persistent
apt-get install nmap # for testing
```

## VLAN

The assumed configuration is a "router" host is connected to a 802.1Q-supporting switch on a "hybrid trunk" port - that is, both untagged (default VLAN) and tagged packets are allowed.

Assuming VLAN ID 1 is the switch's default VLAN, we'll use VLID 2 for the internal network. Using `netplan`, we'll configure the interface to acquire an IP via DHCP from the WAN, and this will be untagged traffic. We'll create a separate virtual interface for the VLID 2 traffic on the same physical port.


In `/etc/netplan/99-config.yaml`:

```yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    vlans:
        vlan2:
            id: 2
            link: eth0
            addresses:
            - 192.168.20.1/24
    version: 2
```

Then apply the configuration:

```
sudo netplan apply
```

If the managed switch being used is configured with another port assigned to VLAN 2, and a "client" device is conncted to that with a (for now) static IP in the `192.168.20.1/24` range, the router and client should be able to communicate with one another.

## DHCP

We don't want to rely on static IPs for client devices; the router should run a separate DHCP server for the VLID 2 VLAN. This will allow us to also manage things like signposting TFTP and DNS too in future.

We can configure a subnet with a range of IPs that can be handed out, in `/etc/dhcp/dhcpd.conf`:

```
default-lease-time 600;
max-lease-time 7200;

subnet 192.168.20.0 netmask 255.255.255.0 {
    range 192.168.20.10 192.168.20.200;
    option routers 192.168.20.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option domain-name "cluster";
}
```

We also have to bind the DHCP server to an interface that already has an IP in the subnet's range, and configure this separately in `/etc/default/isc-dhcp-server`:

```
INTERFACESv4="vlan2"
```

Note - we must refer to the actual device name here (see `sudo cat /proc/net/vlan/config`) rather than the label that netplan assigned (e.g. `vlan2@eth0`).

Now, the client device can be configured for DHCP, and request a new lease. It should be possible to observe the request arriving on the router with by watching the service's logs: `journalctl -u isc-dhcp-server.service -b -f`.

## Routing

While the client can reach the "router", no routing of traffic destined for the WAN is happening; we need to configure that. Firstly, (all on the router), we'll need to enable IP forwarding:

```
# Edit config as necessary to ensure this is set:
$ cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1

# Re-read config:
sudo sysctl --system
```

We then need to add forwarding rules in the filter table:

```
sudo iptables -A FORWARD -i vlan2@eth0 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -o vlan2@eth0 -i eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

...and then enable NAT:

```
sudo iptables -A POSTROUTING -t nat -i vlan2@eth0 -o eth0 -j MASQUERADE
```

We'll then need to ensure these rules survive a reboot:

```
sudo iptables-save > /etc/iptables/rules.v4
```

## DNS

Basic setup as a caching server:

```
sudo apt-get install bind9
```

Tweak the configuration at `/etc/bind/named.conf.options` so it just serves the VLAN:

```diff
- listen-on-v6 { any; };
+ listen-on { 192.168.20.1; };
```

Then modify the DHCP configuration to suggest the router as a DNS server instead:

```diff
-     option domain-name-servers 8.8.8.8, 8.8.4.4;
+     option domain-name-servers 192.168.20.1;
```

If a client is rebooted and a new DHCP lease acquired, BIND should then be used for its DNS lookups - which can be monitored on the router:

```
journalctl -u named.service -b -f
```