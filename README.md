# Notes on VLAN / DHCP / routing setup

## Packages

```
apt-get install vlan isc-dhcp-server iptables-persistent
apt-get install nmap # for testing
```

## VLAN

The assumed configuration is a "router" host is connected to a 802.1Q-supporting switch on a "hybrid trunk" port - that is, both untagged (default VLAN) and tagged packets are allowed.

Assuming VLAN ID 1 is the switch's default VLAN, we'll use VLID 2 for the internal network. Using `netplan`, we'll configure the interface to acquire an IP via DHCP from the WAN, and this will be untagged traffic. We'll create a separate virtual interface for the VLID 2 traffic on the same physical port.

<details>
  <summary>Screenshot of GS308E settings</summary>

  <img width="779" alt="802 1Q" src="https://github.com/joshpencheon/cluster-networking/assets/30904/5b009fed-21df-4ea9-acab-fc5412415a61">
</details>

<details>
  <summary>Screenshots of GS308EPP settings</summary>

  <img width="517" alt="Screenshot 2024-12-15 at 13 56 05" src="https://github.com/user-attachments/assets/f6741594-c427-4120-9dbe-bdb855277ea7" />
  <img width="518" alt="Screenshot 2024-12-15 at 13 53 43" src="https://github.com/user-attachments/assets/d8b28824-8d62-49d4-967f-0ab795949182" />
</details>

-----

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

_⚠️ Note - we must refer to the actual device name here (see `sudo cat /proc/net/vlan/config`) rather than the label that netplan assigned (e.g. `vlan2@eth0`)._

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
sudo iptables -A POSTROUTING -t nat -o eth0 -j MASQUERADE
```

We'll then need to ensure these rules survive a reboot:

```
sudo iptables-save | sudo tee /etc/iptables/rules.v4
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

## TFTP

When netbooting, the boot files can be delivered to the client via TFTP. We'll host a TFTP service on the "router":

```
sudo apt-get install tftpd-hpa
```

The default settings are probably fine, although they can be changed in `/etc/default/tftpd-hpa`. One thing worth changing is the interface to listen on; this defaults to all interfaces, port `:69`. The `TFTP_ADDRESS` configuration can be limited to e.g. single IPv4 address, and then this verified via `ss`:

```bash
sudo ss -pul 'sport = :69'
# State      Recv-Q     Send-Q         Local Address:Port           Peer Address:Port     Process
# UNCONN     0          0               192.168.20.1:tftp                0.0.0.0:*         users:(("in.tftpd",pid=3139,fd=4))
```

To test out, we can install the TFTP client on another machine, and retrieve a file placed in `/srv/tftp`:

```
$ sudo apt-get install tftp-hpa
$ tftp 192.168.20.1
tftp> get test.txt
tftp> quit
$ cat test.txt
```

We'll also update the DHCP subnet config in `/etc/dhcp/dhcpd.conf` to advertise this TFTP service as where clients can get boot config from:

```diff
  range 192.168.20.10 192.168.20.200;
+ option tftp-server-name "192.168.20.1";
  option routers 192.168.20.1;
```

_Note that we're setting [the BOOTP extension option 66](https://www.rfc-editor.org/rfc/rfc2132.html#section-9) directly here, rather than the more generic `next-server` (which doesn't work with Raspberry Pis)._

From the same device, we can use an `nmap` test script to send out a DHCP request onto our VLAN and see the BOOTP option in action:

```
sudo nmap -e vlan2 --script broadcast-dhcp-discover

|   Response 1 of 1:
|     IP Offered: 192.168.20.26
|     DHCP Message Type: DHCPOFFER
|     Server Identifier: 192.168.20.1
|     IP Address Lease Time: 5m00s
|     Subnet Mask: 255.255.255.0
|     Router: 192.168.20.1
|     Domain Name Server: 192.168.20.1
|     Domain Name: cluster
|_    TFTP Server Name: 192.168.20.1
```

## Raspberry PI netboot config

RPi4 support booting from the network, but need some EEPROM flags set. Assuming Ubuntu is being used:

```
sudo apt-get install rpi-eeprom
sudo -E rpi-eeprom-config --edit
```

Update `TFTP_PREFIX` so the mac address is included as a directory prefix (rather than the Pi's serial number), and set the `BOOT_ORDER` so that SD Card (`1`) then netboot (`2`) are tried (the argument is read right-to-left):

```
TFTP_PREFIX=2
BOOT_ORDER=0x21
```

Then reboot to apply the changes, and ensure they stuck by running `rpi-eeprom-config` without arguments.

To test out the start of the netbooting capability, power cycling with the SD card removed should trigger the netboot process, and can be monitored on the router:

```
# a DHCPREQUEST should arrive as per normal:
journalctl -u isc-dhcp-server.service -b -f

# then, boot files should be requested via TFTP:
sudo tcpdump -vv -i vlan2 port 69
```

In the output, we can see e.g. a request for `start4.elf` in a directory named for the client's mac address:

```
192.168.20.27.21979 > pi1.tftp: [udp sum ok] TFTP, length 58, RRQ "xx-xx-xx-xx-xx-xx/start4.elf" octet tsize 0 blksize 1024
```

## Booting a kernel via TFTP

Let's get those boot files available to be served via TFTP. We'll start with a fresh install:

```
wget https://cdimage.ubuntu.com/releases/22.04.3/release/ubuntu-22.04.3-preinstalled-server-arm64+raspi.img.xz
xz -d ubuntu-22.04.3-preinstalled-server-arm64+raspi.img.xz
```

Then check the partitions:

```
fdisk -lu ubuntu-22.04.3-preinstalled-server-arm64+raspi.img

Device                                              Boot  Start     End Sectors  Size Id Type
ubuntu-22.04.3-preinstalled-server-arm64+raspi.img1 *      2048  526335  524288  256M  c W95 FAT32 (LBA)
ubuntu-22.04.3-preinstalled-server-arm64+raspi.img2      526336 8320243 7793908  3.7G 83 Linux
```

Mount the boot partition via a loop device:

```
# get the next free device:
sudo losetup -f

# set up the loop device, using a calculated offset to get the first partition:
sudo losetup -o $((2048*512)) /dev/loop11 ubuntu-22.04.3-preinstalled-server-arm64+raspi.img

# mount it:
sudo mount /dev/loop11 /mnt
```

Now we can copy the firmware necessary to boot into the TFTP area for our client device, identified by mac address:

```
sudo mkdir -p /root/boot
sudo cp -a /mnt /root/boot/ubuntu-22.04.3
sudo cp -a /root/boot/ubuntu-22.04.3 /srv/tftp/xx-xx-xx-xx-xx-xx
```

Finally, clean up the mount and loop device:

```
sudo umount /mnt
sudo losetup -d /dev/loop11
```

Power cycling our client device now should cause it to retrieve the necessary files via TFTP and boot the kernel. However, we've not yet made a root filesystem available.

## Root Filesystem (via NFS)

We now want to make accessible root filesystem. The eventual plan would be to access this via iSCSI, but that needs a custom kernel so we'll start off with NFS instead.

```
sudo apt-get install nfs-kernel-server
```

Extract the root filesystem:

```
sudo losetup -o $((526336*512)) /dev/loop11 ubuntu-22.04.3-preinstalled-server-arm64+raspi.img
sudo mount /dev/loop11 /mnt
sudo mkdir -p /root/filesystems
sudo cp -a /mnt /root/filesystems/ubuntu-22.04.3
```

...again, cleaning up afterwards:

```
sudo umount /mnt
sudo losetup -d /dev/loop11
```

Now make a copy of it for our specific client to serve over NFS:

```
sudo cp -a /root/filesystems/ubuntu-22.04.3 /srv/root/xx-xx-xx-xx-xx-xx
```

We need to make just a couple of tweaks to it given it'll now be arriving via NFS.

Firstly, we'll re-establish the presence of the `/boot/firmware` directory (which initially gets served via TFTP), in order to allow for kernel updates and default `cloud-init` behaviours:

```
sudo mount --bind /srv/tftp/xx-xx-xx-xx-xx-xx /srv/root/xx-xx-xx-xx-xx-xx/boot/firmware
```

Secondly, we'll replace the default filesystem table with one that'll persist this bind mount over subsequent reboots:
```
cat << EOF | sudo tee /srv/root/xx-xx-xx-xx-xx-xx/etc/fstab
/srv/tftp/xx-xx-xx-xx-xx-xx /srv/root/xx-xx-xx-xx-xx-xx/boot/firmware  none   bind   0   0
EOF
```

Now, let's configure the NFS service to export this directory, by editing `/etc/exports` and adding the following:

```
/srv/root/xx-xx-xx-xx-xx-xx *(rw,sync,no_subtree_check,no_root_squash,crossmnt)
```

In detail:
* `*` means this is accessible to all hosts, which is alright for now
* `rw` allows clients to write as well as read ;-)
* `no_subtree_check` is for performance
* `no_root_squash` ensures permissions continue to work in client's FS
* `crossmnt` allows the content of the nested bind mount to be available automatically


Apply the NFS configuration with `sudo exportfs -a`.

You can test this out by temporarily re-mounting via NFS:

```
sudo mount -r -o v3 192.168.20.1:/srv/root/xx-xx-xx-xx-xx-xx /mnt
sudo umount /mnt
```

Now, let's tweak the initial `/srv/tftp/xx-xx-xx-xx-xx-xx/cmdline.txt` to load the root filesystem via NFS:

```diff
- console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=LABEL=writable rootfstype=ext4 rootwait fixrtc quiet splash
+ console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=/dev/nfs nfsroot=192.168.20.1:/srv/root/xx-xx-xx-xx-xx-xx rw rootwait fixrtc quiet splash
```

## Cloud-init

_This section is a bit of wandering digression into trying to get the vanilla 22.04 LTS image booting; in reality, it'd make much more sense to prepare a tweaked image / cloud-init config_

The client should now boot, and we _should_ be able to log in. Unfortunately, we won't be allowed to.

This is because (in the case of Ubuntu 22.04) `cloud-init` is used to provision the OS on first boot, using the `NoCloud` provider. This looks for `user-data` configuration either in the root of specially-labelled partitions, or via command line arguments that were provided to the kernel. Ubuntu use the former method, but rather than labelling the `vfat` partition with the conventional `CIDATA` label it has custom configuration in `/etc/cloud/cloud.cfg.d/99-fake_cloud.cfg` to have `cloud-init` instead look for partitions with `LABEL=system-boot`. This would conventionally match the partition that gets mounted at `/boot/firmware`, where the various config files are indeed found.

However, as we're now exposing the entire filesystem via NFS, we can't rely on this labelling mechanism. Instead, we could try tweaking the kernel commandline again to point at where `cloud-init` config can be found:

```diff
- console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=/dev/nfs nfsroot=192.168.20.1:/srv/root/xx-xx-xx-xx-xx-xx rw rootwait fixrtc quiet splash
+ console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=/dev/nfs nfsroot=192.168.20.1:/srv/root/xx-xx-xx-xx-xx-xx rw rootwait ds=nocloud;s=file:///boot/firmware/ fixrtc quiet splash
```

However, at this point we _still_ have problems. Although `cloud-init` now successfully sources all the required information, the `cloud-config.service` that applies many of the changes gets blocked. After much digging, we find that there are some bugs with AppArmor and having an NFS filesystem, where programs accessing files have unpermitted network access attributed to them. This was observed e.g. with `man`, and also Snaps had problems, meaning the prerequisite `snapd.seeded.service` gets stuck waiting on `snapd.service`, blocking others.

Whilst this is apparently addressed in kernel version 6.0, that's newer than the 22.04 LTS that we're currently using. So for now, we'll just disable AppArmor (!): 

```
- console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=/dev/nfs nfsroot=192.168.20.1:/srv/root/xx-xx-xx-xx-xx-xx rw rootwait ds=nocloud;s=file:///boot/firmware/ fixrtc quiet splash
+ console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=/dev/nfs nfsroot=192.168.20.1:/srv/root/xx-xx-xx-xx-xx-xx rw rootwait ds=nocloud;s=file:///boot/firmware/ fixrtc quiet splash apparmor=0
```

## Adding Wi-Fi

We can use the Wi-Fi antenna in the router RPi to servce as a dedicated wireless access point for the cluster's VLAN. First, install the necessary package:

```
sudo apt-get install hostapd
```

Then add some configuration to `/etc/hostapd/hostapd.conf`:

```
interface=wlan0
bridge=br0

# 2.4 GHz:
hw_mode=b
channel=1
macaddr_acl=0
auth_algs=1
country_code=GB
driver=nl80211
ignore_broadcast_ssid=0

# Only support WPA2 with AES:
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP

ssid=YourNetworkName
wpa_passphrase=YourNetworkPassphrase
```

We'll need to point at this configuration, by uncommenting the following in `/etc/default/hostapd`:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

We can now unmask (I think it is masked by default to ensure you configure radio settings, perhaps?), enable and start the service:

```
sudo systemctl unmask hostapd.service
sudo systemctl enable hostapd.service
sudo systemctl start hostapd.service
```

At this point, a device should be able to see the network, but not get a proper connection to it. We'll need to start off by properly bridging the WLAN interface with our VLAN - using `netplan`, we'll create a bridge that inherits our former VLAN config. In `/etc/netplan/99-config.yaml`:

```
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: true
        wlan0:
            dhcp4: false
    vlans:
        vlan2:
            id: 2
            link: eth0
    bridges:
        br0:
            interfaces:
            - wlan0
            - vlan2
            addresses:
            - 192.168.20.1/24
    version: 2
```

After this configuration has been applied via `netplan`, we can update our `iptables` rules. This is most easily done just by editing `/etc/iptables/rules.v4` and then using `iptables-restore` to flush out the old tables and apply the new ones. We'll update the incoming and outgoing rules to refer to `br0` instead of `vlan2@eth0`:

```
-A FORWARD -i br0 -o eth0 -j ACCEPT
-A FORWARD -i eth0 -o br0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Finally, we'll need to correct our DHCP configuration to have that listen on the bridged interface instead. In `/etc/default/isc-dhcp-server`:

```
INTERFACESv4="br0"
```

Once the DHCP service has been restarted, a device connecting to the new Wi-Fi network should receive an IP address from the `192.168.20.0/24` range (and show up in `dhcp-lease-list`). Be warned that the RPi's radio isn't the fastest by any means.

## TODO

* Kernel updates
* Investigate static IPs + hostnames from DHCP, to avoid bootloader consuming an extra IP
* NFS access controls?
* unionFS of root filesystems?
* iSCSI
* Pi5 NVMe setup
* Ansible
  

## Project Ideas

* k3s
* kamal
* ceph
* litestream / rqlite
* time machine target
