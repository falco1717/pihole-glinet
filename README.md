# Install Package on external storage
```
opkg update
opkg install fdisk e2fsprogs
```
Insert USB drive

SSH into GlInet
Find drive location
```
df -h
```
Probably /dev/sda1
```
umount /dev/sda1
mkfs.ext4 /dev/sda1
```
edit /etc/profile, add below lines to existing export lines. Replace sdb1 in the first line to yours, per your df output.
```
export USB=/mnt/sda1
export PATH=$PATH:$USB/usr/bin:$USB/usr/sbin # This PATH is dependent on existing $PATH
export LD_LIBRARY_PATH=$USB/lib:$USB/usr/lib
```
edit /etc/opkg.conf, add one line:
```
dest usb /mnt/sda1
```
reboot
# Create Symbolic Links
```
mkdir /tmp/mountd/disk1_part1/docker
mkdir /tmp/mountd/disk1_part1/containerd
ln -s /tmp/mountd/disk1_part1/docker /opt/docker
ln -s /tmp/mountd/disk1_part1/containerd /opt/containerd
```
# Download and Install Docker/MacVlan packages
```
opkg update && opkg install docker luci-app-dockerman docker-compose dockerd kmod-macvlan nano git git-http
reboot
```
# Setup MACVLAN interface in OpenWrt
Here the MACVLAN interface will be created which is a virtual interface bridged to current LAN interface. This interface will later be used by Docker and PiHole.
```
nano /etc/config/network
```
Look for the `config interface 'lan'` section and note down the device from the `option device` line. In my case it is `br-lan`.

Add the following at the end of the file. The key part here is `option device` and `option name` which is the same device as in the lan section with `.20` suffix.
```
config interface 'macvlan'
        option proto 'static'
        option defaultroute '0'
        option netmask '255.255.255.255'
        option device 'br-lan.20'
        option ipaddr '192.168.8.2'

config device
        option type 'macvlan'
        option ifname 'br-lan'
        option mode 'bridge'
        option name 'br-lan.20'
        option acceptlocal '1'

config route
        option interface 'macvlan'
        option target '192.168.8.3'
        option netmask '255.255.255.255'
```
# Modify the firewall adding the new interface to the lan zone
```
nano /etc/config/firewall
```
Look for section similar to below:
```
config zone
        option name 'lan'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'ACCEPT'
        list network 'lan'
```
Then add the following line to it:
```
list network 'macvlan'
```
# Restart the network and firewall services
```
/etc/init.d/network restart 
/etc/init.d/firewall restart
```
# Setup docker-compose.yml and PiHole

Create the folders for the volumes:
```
mkdir -p /tmp/mountd/disk1_part1/docker/pihole/etc/pihole/
mkdir -p /tmp/mountd/disk1_part1/docker/pihole/etc/dnsmasq.d/
mkdir -p /tmp/mountd/disk1_part1/docker/pihole/var-log/
mkdir -p /tmp/mountd/disk1_part1/docker/pihole/var-log/lighttpd
chown 33:33 /tmp/mountd/disk1_part1/docker/pihole/var-log/lighttpd
mkdir -p /tmp/mountd/disk1_part1/docker/pihole/etc-cont-init.d/
```
# Download Github Repo
```
git clone https://github.com/falco1717/pihole-glinet /tmp/mountd/disk1_part1/docker/
```

Create 10-fixroutes.sh.
```
echo '#!/usr/bin/with-contenv bash
set -e

echo "fixing routes"
ip route del default
ip route add default via 172.18.0.1
echo "done fixing routes"' >> /tmp/mountd/disk1_part1/docker/pihole/etc-cont-init.d/10-fixroutes.sh
chmod 755 ./pihole/etc-cont-init.d/10-fixroutes.sh
```
# Start PiHole and finalize its setup
```
cd /tmp/mountd/disk1_part1/docker
docker-compose up -d pihole
docker logs -f pihole
```
Restart PiHole container
```
docker restart pihole
```
# Test
You could do some testing and see if it works. From another machine connected to the LAN network or WiFi:
```
ping 192.168.8.3
nslookup pihole.lan 192.168.8.3
nslookup openwrt.org 192.168.8.3
# For *nix
nslookup $(hostname).lan 192.168.8.3
# For Windows
nslookup %COMPUTERNAME%.lan 192.168.8.3
```
Open PiHole admin: http://pihole.lan/admin 85

If all of the above works then PiHole container is ready to be the DNS server for your devices on the LAN network.

# Configure DNS settings in OpenWrt and Profit!
```
nano /etc/config/dhcp
```
Look for the following section:
```
config dhcp 'lan'
        option interface 'lan'
        option start '100'
        option limit '150'
```
Then add the following line to it:
```
list dhcp_option '6,192.168.8.3'
```
Restart dnsmasq service (the service that manages the DHCP on openwrt):
```
/etc/init.d/dnsmasq restart
```
# Change GLinet DNS Server
https://192.168.8.1/#/dnsview
Change mode to manual
```
enter 192.168.8.3
```
