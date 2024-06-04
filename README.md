# Download and Install Docker/MacVlan packages
```
opkg update && opkg install docker luci-app-dockerman docker-compose dockerd kmod-macvlan nano
reboot
```
# Setup MACVLAN interface in OpenWrt
Here the MACVLAN interface will be created which is a virtual interface bridged to current LAN interface. This interface will later be used by Docker and PiHole.
```
vi /etc/config/network
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
```
cd ~
nano docker-compose.yml
```
Copy and paste the following and save. The following was retrieved from PiHole docs and slightly modified to fulfill our purpose. In the netowrk section at the bottom, the parent is set to the new macvlan interface created in the previous step from the option device line (the one with suffix .20). Also, the PiHole container will have a static IP which I've set to 192.168.8.3
```
version: '3'

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    environment:
      TZ: 'America/Los_Angeles'  # Adjust timezone as needed
      WEBPASSWORD: 'Your Password'
    volumes:
      - '/tmp/mountd/disk1_part1/docker/pihole/etc/pihole/:/etc/pihole/'
      - '/tmp/mountd/disk1_part1/docker/pihole/etc/dnsmasq.d/:/etc/dnsmasq.d/'
    networks:
      lan:
        ipv4_address: 192.168.8.3
    restart: unless-stopped
    cap_add:
      - NET_ADMIN

networks:
  lan:
    external:
      name: macvlan_lan
```
Create the folders for the volumes:
```
mkdir -p ./pihole/etc-pihole/
mkdir -p ./pihole/etc-dnsmasq.d/
mkdir -p ./pihole/var-log/
mkdir -p ./pihole/var-log/lighttpd
chown 33:33 ./pihole/var-log/lighttpd
mkdir -p ./pihole/etc-cont-init.d/
```
Create 10-fixroutes.sh.
```
echo '#!/usr/bin/with-contenv bash
set -e

echo "fixing routes"
ip route del default
ip route add default via 172.18.0.1
echo "done fixing routes"' >> ./pihole/etc-cont-init.d/10-fixroutes.sh
chmod 755 ./pihole/etc-cont-init.d/10-fixroutes.sh
```
# Start PiHole and finalize its setup
```
cd ~
docker-compose up -d pihole
docker logs -f pihole
```
Wait for PiHole to finish starting up and then hit CTRL+C. Then run the following:
```
cd ~/pihole/etc-pihole
sed -i -e 's/REV_SERVER.*//; s/REV_SERVER_CIDR.*//; s/REV_SERVER_TARGET.*//; s/REV_SERVER_DOMAIN.*//; s/PIHOLE_INTERFACE.*//' setupVars.conf
echo 'REV_SERVER=true
REV_SERVER_CIDR=192.168.1.0/24
REV_SERVER_TARGET=127.0.0.11
REV_SERVER_DOMAIN=lan
PIHOLE_INTERFACE=eth0' >> setupVars.conf
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
