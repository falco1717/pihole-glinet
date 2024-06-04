Disable IPv6 DHCP on OpenWrt
```uci -q delete dhcp.lan.dhcpv6
uci -q delete dhcp.lan.ra
uci -q delete network.lan.ipv6
uci commit
/etc/init.d/odhcpd stop
/etc/init.d/odhcpd disable
/etc/init.d/network restart```

![image](https://github.com/falco1717/pihole---openwrt/assets/74680727/cff0f9f1-0bda-4aa7-8367-b42ad654ceb6)


Install Docker and Docker-Compose on OpenWrt
```opkg update && opkg install docker luci-app-dockerman docker-compose dockerd kmod-macvlan
reboot```
