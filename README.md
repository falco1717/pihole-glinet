Disable IPv6 DHCP on OpenWrt
```
uci -q delete dhcp.lan.dhcpv6
uci -q delete dhcp.lan.ra
uci -q delete network.lan.ipv6
uci commit
/etc/init.d/odhcpd stop
/etc/init.d/odhcpd disable
/etc/init.d/network restart
```
Download and Install Docker/MacVlan packages
```
opkg update && opkg install docker luci-app-dockerman docker-compose dockerd kmod-macvlan
reboot
```
Here the MACVLAN interface will be created which is a virtual interface bridged to current LAN interface. This interface will later be used by Docker and PiHole.
```
vi /etc/config/network
```
Look for the `config interface 'lan'` section and note down the device from the `option device` line. In my case it is `br-lan`.

Add the following at the end of the file. The key part here is `option device` and `option name` which is the same device as in the lan section with `.20` suffix.
