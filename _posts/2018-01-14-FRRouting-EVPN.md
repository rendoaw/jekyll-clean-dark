---
layout: post
comments: true
title: "FRRouting and EVPN - part 1"
categories: blog
descriptions: First experiment with FRRouting and EVPN
tags: 
  - docker
  - networking
  - FRRouting
date: 2018-01-14T10:39:55-04:00
---


This is the first post related to FRR Routing test, mainly anything related to EVPN or L2/L3 VPN in general. As starting point, the following note shows the config example for basic EVPN type 2 and it was tested to work with Cisco Nexus and Arista EOS. 

## Test Summary

### Lab environment

* Basic leaf-spine IP fabric
	* vMX as spine and RR
	* vEOS, Nexus 9Kv and FRRouting as leaf
		* FRRouting run as docker inside Ubuntu 16.04 linux VM
* Traffic type
	* L2 stretch with EVPN type 2  
	* "L3 VPN" with EVPN type 5

### Result

* L2 stretch EVPN type 2 works between FRR, NXOS and EOS.
	* IRB routing with EVPN-type 2 is not tested yet
	* vlan-based evpn is tested. Vlan aware is not tested yet. 
* EVPN type 5
	* It seems that FRR can receive EVPN type 5 routes (see the sample output section below)
	* But, FRR does not advertise anything. 
		* "advertise-subnet" command as indicated in Cumulus Linux guide is not available in this FRR version (build based on FRR github master branch as of 2018-01-14)




## Details

### (Optional) upgrade kernel to 4.8 

This is optional for later testing, e.g: setup vrf, run mpls/ldp. Not really needed in this post. 

```
apt-get -y install linux-image-4.8.0-41-generic linux-image-extra-4.8.0-41-generic
```



### Update sysctl config to allow forwarding

"net.ipv4.ip_forward = 1" and "net.ipv6.conf.all.forwarding=1" should be good enough for this test, but no harm to implement the whole sysctl setting as recommended by FRRouting.

Put the following into /etc/sysctl.d/99frr_defaults.conf

```
# /etc/sysctl.d/99frr_defaults.conf
# Place this file at the location above and reload the device.
# or run the sysctl -p /etc/sysctl.d/99frr_defaults.conf

# Enables IPv4/IPv6 Routing
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding=1

# Routing
net.ipv6.route.max_size=131072
net.ipv4.conf.all.ignore_routes_with_linkdown=1
net.ipv6.conf.all.ignore_routes_with_linkdown=1

# Best Settings for Peering w/ BGP Unnumbered
#    and OSPF Neighbors
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.lo.rp_filter = 0
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.default.forwarding = 1
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.default.arp_notify = 1
net.ipv4.conf.default.arp_ignore=1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.all.arp_notify = 1
net.ipv4.conf.all.arp_ignore=1
net.ipv4.icmp_errors_use_inbound_ifaddr=1

# Miscellaneous Settings

#   Keep ipv6 permanent addresses on an admin down
net.ipv6.conf.all.keep_addr_on_down=1

# igmp
net.ipv4.igmp_max_memberships=1000
net.ipv4.neigh.default.mcast_solicit = 10

# MLD
net.ipv6.mld_max_msf=512

# Garbage Collection Settings for ARP and Neighbors
net.ipv4.neigh.default.gc_thresh2=7168
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv4.neigh.default.base_reachable_time_ms=14400000
net.ipv6.neigh.default.gc_thresh2=3584
net.ipv6.neigh.default.gc_thresh3=4096
net.ipv6.neigh.default.base_reachable_time_ms=14400000

# Use neigh information on selection of nexthop for multipath hops
net.ipv4.fib_multipath_use_neigh=1

# Allows Apps to Work with VRF
net.ipv4.tcp_l3mdev_accept=1
```


### Reboot the host

In theory, we can apply the new rule by using sysctl -p ..., but it happened to me that for some reason, the bridge interface does not forward traffic between interface until i reboot the host. No idea why.



### setup the network 

* setup vxlan interface(s), 1 vxlan per vni

In this example i have vni 10100. I created vxlan100 because the this vni correspon to vlan 100 on the other side. It just my numbering scheme.

```
ip link add vxlan100 type vxlan id 10100 dstport 4789 local 10.0.0.94 
ip link set dev vxlan100 up
```


* setup IGP link

In this example, i have ens5 interface connected to spine switch. I will enable ospf on this interface when i configure the FRR. 

```
ip a add 10.11.94.2/30 dev ens5
ip link set dev ens5 up
```

* configure loopback IP

```
ip a add 10.0.0.94/32 dev lo
ip link set dev lo up
```


* Setup CE / host facing interface and the bridge

In this example, i have ens6 interface connected to the vxlan CE. I created bridge br100 to connect vxlan tunnel (vxlan100) to the physical interface (ens6)

```
ip link set dev ens6 up
brctl addbr br100
brctl addif br100 vxlan100
brctl addif br100 ens6
ip a add 10.21.100.94/24 dev br100
ip link set dev br100 up
```



### Run docker image

In this case, because i want to use FRRouting inside docker as the same as if i run natively on the linux host, so I put the container with -net=host. To make it simple, i also provide full proviledge to the container.


```
# docker run -d -it --net=host --name frr  --privileged --restart=always rendoaw/docker_frr
```


### Attach to docker container

There are 2 options. The first option is to attach directly to FRRouting vty shell as shown below. The other option is to attach to bash shell and run vtysh from there. The later option is usefull if we need to modify any FRRouting setting, for example enable/disable some daemons.

```
root@linux-94:/home/ubuntu# docker exec -it frr vtysh

Hello, this is FRRouting (version 3.1-devyes-gb782607).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

This is a git build of frr-3.1-dev-680-gb782607
Associated branch(es):
    local:master
    github/frrouting/frr.git/master

linux-94#
```



### (Reference) FRRouting config and show output examples


* Sample EVPN config

```
root@linux-94:/home/ubuntu# docker exec -it frr vtysh

Hello, this is FRRouting (version 3.1-devyes-gb782607).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

This is a git build of frr-3.1-dev-680-gb782607
Associated branch(es):
    local:master
    github/frrouting/frr.git/master

linux-94# sh run
Building configuration...

Current configuration:
!
frr version 3.1-devyes
frr defaults traditional
hostname linux-94
username cumulus nopassword
!
service integrated-vtysh-config
!
log syslog informational
!
vrf vrf-110
 vni 20110
!
interface ens5
 ip ospf network point-to-point
!
router-id 10.0.0.94
!
router bgp 64000
 coalesce-time 1000
 neighbor 10.0.0.11 remote-as 64000
 neighbor 10.0.0.11 update-source 10.0.0.94
 !
 address-family l2vpn evpn
  neighbor 10.0.0.11 activate
  vni 20110
   rd 10.0.0.94:110
   route-target import 64000:110
   route-target export 64000:110
   advertise-default-gw
  exit-vni
  vni 10100
   rd 10.0.0.94:100
   route-target import 64200:100
   route-target export 64200:100
  exit-vni
  advertise-all-vni
  advertise ipv4 unicast
 exit-address-family
 vrf-policy test
  exit-vrf-policy
 vnc defaults
  exit-vnc
!
router bgp 64000 vrf vrf-110
 coalesce-time 1000
 !
 address-family ipv4 unicast
  network 0.0.0.0/0
 exit-address-family
!
router ospf
 network 10.0.0.0/24 area 0
 network 10.11.0.0/16 area 0
!
line vty
!
end
linux-94#
```


* Sample show output

```
linux-94# show ip ospf  neighbor

Neighbor ID     Pri State           Dead Time Address         Interface
   RXmtL RqstL DBsmL
10.0.0.11       128 Full/DROther      38.688s 10.11.94.1      ens5:10.11.94.2
       0     0     0

linux-94# show bgp l2vpn evpn summary
BGP router identifier 10.0.0.94, local AS number 64000 vrf-id 0
BGP table version 0
RIB entries 25, using 3800 bytes of memory
Peers 1, using 20 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/P
fxRcd
10.0.0.11       4      64000     531     308        0    0    0 01:25:50
   20

Total number of neighbors 1
linux-94# show bgp l2vpn evpn route
BGP table version is 0, local router ID is 10.0.0.94
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 10.0.0.11:100
*>i[2]:[0]:[0]:[48]:[00:05:86:2b:d6:f0]
                    10.0.0.11                     100      0 i
*>i[2]:[0]:[0]:[48]:[00:05:86:2b:d6:f0]:[32]:[10.21.100.11]
                    10.0.0.11                     100      0 i
*>i[3]:[0]:[32]:[10.0.0.11]
                    10.0.0.11                     100      0 i
Route Distinguisher: 10.0.0.21:110
*>i[5]:[0]:[24]:[10.21.121.0]
                    10.0.1.21                0    100      0 ?
Route Distinguisher: 10.0.0.21:32867
*>i[3]:[0]:[32]:[10.0.1.21]
                    10.0.1.21                     100      0 i
Route Distinguisher: 10.0.0.22:100
*>i[2]:[0]:[0]:[48]:[52:55:00:02:11:19]
                    10.0.0.22                     100      0 i
*>i[3]:[0]:[32]:[10.0.0.22]
                    10.0.0.22                     100      0 i
Route Distinguisher: 10.0.0.22:200
*>i[2]:[0]:[0]:[48]:[52:55:00:02:11:19]
                    10.0.0.22                     100      0 i
*>i[3]:[0]:[32]:[10.0.0.22]
                    10.0.0.22                     100      0 i
Route Distinguisher: 10.0.0.23:110
*>i[5]:[0]:[24]:[10.23.123.0]
                    10.0.1.23                0    100      0 ?
Route Distinguisher: 10.0.0.23:200
*>i[3]:[0]:[32]:[10.0.1.23]
                    10.0.1.23                     100      0 i
Route Distinguisher: 10.0.0.25:100
*>i[2]:[0]:[0]:[48]:[8e:15:27:dd:79:0a]
                    10.0.0.25                     100      0 i
*>i[2]:[0]:[0]:[48]:[8e:15:27:dd:79:0a]:[32]:[10.21.100.125]
                    10.0.0.25                     100      0 i
*>i[3]:[0]:[32]:[10.0.0.25]
                    10.0.0.25                     100      0 i
Route Distinguisher: 10.0.0.25:110
*>i[5]:[0]:[24]:[10.25.125.0]
                    10.0.0.25                     100      0 i
Route Distinguisher: 10.0.0.27:2
*>i[2]:[0]:[0]:[48]:[52:55:01:00:27:00]:[32]:[10.21.100.27]
                    10.0.0.27                     100      0 i
*>i[2]:[0]:[0]:[48]:[52:55:01:00:27:00]:[128]:[fe80::5055:1ff:fe00:2700]
                    10.0.0.27                     100      0 i
*>i[3]:[0]:[32]:[10.0.0.27]
                    10.0.0.27                     100      0 i
Route Distinguisher: 10.0.0.93:100
*>i[2]:[0]:[0]:[48]:[8e:e4:2a:e4:51:ad]
                    10.0.0.93                     100      0 i
*>i[3]:[0]:[32]:[10.0.0.93]
                    10.0.0.93                     100      0 i
Route Distinguisher: 10.0.0.94:100
*> [2]:[0]:[0]:[48]:[ae:6c:61:26:ef:ea]
                    10.0.0.94                          32768 i
*> [3]:[0]:[32]:[10.0.0.94]
                    10.0.0.94                          32768 i

Displayed 22 prefixes (22 paths)
linux-94#
```


* Sample tcpdump output

	* TCPDUMP from physical interface  (encapsulated packet)

	```
	# tcpdump -e -s 0 -nv -i ens5 port 4789

	ae:6c:61:26:ef:ea > 8e:15:27:dd:79:0a, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 26385, offset 0, flags [DF], proto ICMP (1), length 84)
		10.21.100.194 > 10.21.100.125: ICMP echo request, id 24417, seq 80, length 64
	02:50:09.041530 52:55:00:02:11:22 > 52:55:01:00:94:01, ethertype IPv4 (0x0800), length 148: (tos 0x0, ttl 63, id 0, offset 0, flags [DF], proto UDP (17), length 134)
		10.0.0.25.25167 > 10.0.0.94.4789: VXLAN, flags [I] (0x08), vni 10100
	8e:15:27:dd:79:0a > ae:6c:61:26:ef:ea, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 23894, offset 0, flags [none], proto ICMP (1), length 84)
		10.21.100.125 > 10.21.100.194: ICMP echo reply, id 24417, seq 80, length 64
	```


	* TCPDUMP (decapsulated packet)

	```
 	# tcpdump -e -s 0 -nv -i br100

	02:51:32.095689 ae:6c:61:26:ef:ea > 8e:15:27:dd:79:0a, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 36301, offset 0, flags [DF], proto ICMP (1), length 84)
		10.21.100.194 > 10.21.100.125: ICMP echo request, id 24417, seq 163, length 64
	02:51:32.126877 8e:15:27:dd:79:0a > ae:6c:61:26:ef:ea, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 34235, offset 0, flags [none], proto ICMP (1), length 84)
		10.21.100.125 > 10.21.100.194: ICMP echo reply, id 24417, seq 163, length 64	
	```
