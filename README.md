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

