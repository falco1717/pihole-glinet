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
