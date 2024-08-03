# Openwrt-IPTV-Freedom.nl

Configuration for Openwrt to access Internet and IPTV access by freedom.nl

## Device info

| Spec             | Info          |
| ---------------- | ------------------ |
| Model            | Linksys WRT3200ACM |
| Architecture     | ARMv7 Processor   |
| Target Platform  | mvebu/cortexa9    |
| Firmware version | OpenWrt 23.05.4 r24012-d8dd03c46f / LuCI openwrt-23.05 branch git-24.086.45142-09d5a38 |
| Kernel version   | 5.15.162  |

## Network setup 1#

This setup is configured as simple as possible.

Modem box provided by ISP, connected to WAN port on the Linksys router.
On LAN 3 of the router I have connected

The VLANs that are needed:
|||
|---|---|
| wan.6 | Internet access |
| wan.4 | IPTV access (Enabled multicast support) |
| eth0.7 | Route the IPTV traffic through here to LAN 3 |
| eth0.1 | Default vlan |

Additional bridges configured:

1. `br-lan`:

   - `eth0.1`, `LAN 1`, `LAN 2`, `LAN 3`, `LAN 4`
   - Enable IGMP snooping: `Enabled`
   - Enable multicast support: `Disabled`

   Connects all ports to be able to have internet access.

   Normal interfaces were getting flooded till I enabled those. Not sure which of the two helped or if both are needed.

2. `br-tv`:

   - `eth0.7`, `LAN 3`.
   - Enable IGMP snooping: Disabled
   - Enable multicast support: Enabled

   In order for the device connected to LAN 3 to receive IPTV data as untagged.

### Interfaces

| Name | Protocol | IPv4 | Netmask | Device | Extra opt |
| ---  | -----    | -----|---------|--------|-----------|
|InIPTV| static   | 10.10.33.1 (Should not matter) | 255.255.255.0 | br-tv | Setup DHCP server, default settings |
| IPTV| DHCP client| - | - | wan.4 | Vender class: IPTV_RG (probably does not have an influence) |
|lan|static|192.168.1.1|-|br-lan|-|
|wan|PPPoe|-|-|wan.6|username/password are set|


Unmentioned settings are default in luci. Extra settings are defined in cli.

### Firewall

| Name | Input | Output | Forward | MASQ | Allow forward dest | Allow forward src |
|----|----|----|----|----|----|---|
|lan|accept|accept|accept|-|wan|-|
|wan|accept|accept|reject|yes|-|InIPTV, lan|
|iptv|accept|accept|reject|yes|-|InIPTV|
|InIPTV|accept|accept|accept|-|iptv, wan|-|

#### Extra traffic rules

The rules below are most likely not needed and will be retested.

`Allow ICMP`: Forwarded IPv4, protocol ICMP, from wan to any zone.

`Allow-ISAKMP-IPTV`: Forwarded IPv4 and IPv6, protocol UDP, from iptv to InIPTV, port 500.

`Allow-IPSEC-IPTV`: Forwarded IPv4 and IPv6, protocol IPSEC-ESP, from iptv to InIPTV.

### Settings that cannot be done in luci

ssh to openwrt and follow the steps below.

Install igmpproxy:

```
opkg update
opkg install igmpproxy
```

Modify file:
`/etc/config/igmpproxy`

```
config igmpproxy
	option quickleave 1

config phyint
	option network IPTV
	option zone iptv
	option direction upstream
	list altnet 0.0.0.0/0 # Too permissive. Can narrow it down
#	list altnet 185.24.175.0/24
#	list altnet 185.41.48.0/24
#	list altnet 217.166.0.0/16
#	list altnet 10.10.0.33/32

config phyint
	option network INIPTV
	option zone InIPTV
	option  direction downstream
```

Modify file:
`/etc/config/network`

Edit interface IPTV

```
config interface 'IPTV'
	option proto 'dhcp'
	option hostname 'IPTV-Device'
	option device 'wan.4'
	option peerdns '0' # Need retest to see effect
	option type 'bridge'
	option vendorid 'IPTV_RG' # Need retest to see effect
	option defaultroute '0'
	option sendopts '121' # Required otherwise you only get on demand videos
```

Restart services to apply changes.

```
/etc/init.d/network restart
/etc/init.d/igmpproxy restart
```

`logread` command can help check more logs for debugging.

## Network setup 2#, Include Switch

WIP: Assumption is you can tag LAN 3 with VLAN ID 7 then let the switch handle multicast and delivering it to the correct port.
