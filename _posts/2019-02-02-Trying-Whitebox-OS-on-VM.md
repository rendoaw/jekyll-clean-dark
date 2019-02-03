---
layout: post
comments: true
title: "Trying Whitebox OS on VM"
categories: blog
descriptions: Trying multiple whitebox OS on VM
tags: 
  - onie
  - whitebox
  - openswitch
  - opx
  - cumulus
  - sonic
date: 2019-02-02T10:39:55-04:01
---



This post will show you how to run some whitebox network operating system on VM. This setup includes Microsoft Sonic, Openswitch (OPX), Cumulus VX, Arista vEOS. The main focus of this post will be Sonic and OPX. vEOS and Cumulus VX are included to reduce the unknown factor. We have been testing vEOS and Cumulus VX many times, so we know how to configure them correctly, and if something not working then most likely we have incorrect configuration on the Sonic or OPX side.


## Topology

```
     ┌─────────────┐                                          ┌─────────────┐       
     │             │  port 03                         port 03 │             │       
     │  Sonic-35   │──────────────────────────────────────────│ Cumulus-38  │       
     │ 10.96.0.35  │ 10.35.38.0                    10.35.38.1 │ 10.96.0.38  │       
     └─────────────┘                                          └─────────────┘       
            ╲  port 01                                       port 01 ╱  port 02     
    port 02 │╲ 10.34.35.1                                 10.34.38.1 │  10.36.38.1  
 10.35.37.0 │ ╲──────────────────────╲     ╱───────────────────────╱ │              
            │                         ╲                              │              
            │                 port 01  ╲ ╱ port 02                   │              
            │              10.34.35.0   V  10.34.38.0                │              
            │                    ┌─────────────┐                     │              
            │                    │             │                     │              
            │                    │   vEOS-34   │                     │              
            │                    │  10.96.0.34 │                     │              
            │                    └─────────────┘                     │              
            │               10.34.37.0  Λ   10.34.36.0               │              
            │                  port 03 ╱ ╲  port 04                  │              
            │                             ╲                          │              
            │  ╱─────────────────────╱     ╲──────────────────────╲  │              
            │ ╱                                                    ╲ │              
 10.35.37.1 │╱  10.34.37.1                               10.34.36.1 ╲│   10.36.38.0 
    port 02 ╱  port 01                                      port 01  ╲   port 02    
     ┌─────────────┐                                          ┌─────────────┐       
     │             │ 10.36.37.1                   10.36.37.0  │             │       
     │   OPX-37    │──────────────────────────────────────────│  onie-36    │       
     │ 10.96.0.37  │ port 03                          port 03 │ 10.96.0.36  │       
     └─────────────┘                                          └─────────────┘       
```


* Why VM?
Yes, for real testing, especially data plane testing, we should do it on real whitebox switches, but in my case, i only have very limited number of them. VM is good enough to familiarize with each OS setting and its way of work. VM is also useful for testing and developing any script and any other automation that need to be done for this OS.

* why EOS?
vEOS is included because it is possible to install EOS on whitebox switch. So EOS still qualified as whitebox OS.
Reference: https://github.com/aristanetworks/arista-onie-installer

* What is onie-36 in the topology?
It is a non-free OS like Cumulus, but for now let's call it as "sonie-36"


## VM Deployment

* Sonic
  * Use the ONIE image created by [Install ONIE on VM](Install-ONIE-on-VM)
  * Use onie-nos-install or install_url command to install Sonic binary
  * Sonic binary image for VM can be downloaded from here: [Sonic Onie VM Image](https://sonic-jenkins.westus2.cloudapp.azure.com/job/vs/job/buildimage-vs-image-frr/lastSuccessfulBuild/artifact/target/sonic-vs.bin)
  
* Openswitch(OPX)
  * Use the ONIE image created by [Install ONIE on VM](Install-ONIE-on-VM)
  * Use onie-nos-install or install_url command to install OPX binary
  * OPX binary image for VM can be downloaded from here:
    * [OPX image](http://archive.openswitch.net/installers/3.1.0/Dell-EMC/PKGS_OPX-3.1.0-installer-x86_64.bin)
    * [Openswitch Website](http://archive.openswitch.net/)
  * OPX binary image does not include any routing protocol daemon. We need to install Frrouting/Quagga, BIRD or any other daemon separately
  * for now, to make it easier, i am using FRRouting as well for this OPX.
  
* Cumulus VX
  * There are two option. The first and easy way is to use ready to use Cumulus VX image from [Cumulus VX download](https://cumulusnetworks.com/products/cumulus-vx/)
  * 2nd option, is using cumulus VX onie image, for example: [Cumulus VX Onie Image](https://cumulusnetworks.com/products/cumulus-vx/download/thanks/onie-372-/)
  
  
* vEOS
  * Although we can install EOS on some 3rd party whitebox switch, it still not easy to install it on VM with ONIE. For now, we use the ready to use vEOS image from [Arista EOS Download](https://www.arista.com/en/support/software-download)
  
  
## Some outputs

### Sonic

* config

```
root@sonic-35:/home/admin# cat /etc/sonic/config_db.json
{
    "BGP_NEIGHBOR": {
        "10.34.35.0": {
            "rrclient": 0,
            "name": "veos-34",
            "local_addr": "10.34.35.1",
            "nhopself": 0,
            "holdtime": "180",
            "asn": "65034",
            "keepalive": "60"
        },
        "10.35.37.1": {
            "rrclient": 0,
            "name": "opx-37",
            "local_addr": "10.35.37.0",
            "nhopself": 0,
            "holdtime": "180",
            "asn": "65037",
            "keepalive": "60"
        },
        "10.35.38.1": {
            "rrclient": 0,
            "name": "cumulus-38",
            "local_addr": "10.35.38.0",
            "nhopself": 0,
            "holdtime": "180",
            "asn": "65038",
            "keepalive": "60"
        }
    },
    "DEVICE_METADATA": {
        "localhost": {
            "hwsku": "Force10-S6000",
            "hostname": "sonic-35",
            "platform": "x86_64-kvm_x86_64-r0",
            "mac": "0c:55:01:00:35:00",
            "bgp_asn": "65035",
            "type": "LeafRouter"
        }
    },
    "DEVICE_NEIGHBOR": {},
    "LOOPBACK_INTERFACE": {
        "Loopback0|10.96.0.35/32": {}
    },
    "INTERFACE": {
        "Ethernet0|10.34.35.1/31": {},
        "Ethernet4|10.35.37.0/31": {},
        "Ethernet8|10.35.38.0/31": {}
    },
    "MGMT_INTERFACE": {
        "eth0|192.168.1.35/24": {
            "gwaddr": "192.168.1.1"
        }
    },
    "PORT": {
        "Ethernet0": {
            "admin_status": "up",
            "lanes": "1",
            "mtu": "9100"
        },
        "Ethernet4": {
            "admin_status": "up",
            "lanes": "2",
            "mtu": "9100"
        },
        "Ethernet8": {
            "admin_status": "up",
            "lanes": "3",
            "mtu": "9100"
        }
    }
}
```

* bgp status

```
root@sonic-35:/home/admin# vtysh

Hello, this is FRRouting (version 6.0.2-20190116-00-g5a35fd3).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

sonic# show ip bgp summary

IPv4 Unicast Summary:
BGP router identifier 10.96.0.35, local AS number 65035 vrf-id 0
BGP table version 25
RIB entries 7, using 1120 bytes of memory
Peers 3, using 62 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
10.34.35.0      4      65034     511     515        0    0    0 08:03:35            3
10.35.37.1      4      65037     432     440        0    0    0 01:52:09            3
10.35.38.1      4      65038    4029    4030        0    0    0 02:16:32            2

Total number of neighbors 3
```


### OPX

* config - interfaces

```
root@opx-37:/home/admin# cat /etc/network/interfaces.d/dataport
auto e101-001-0
allow-hotplug e101-001-0
iface e101-001-0 inet static
  netmask 255.255.255.254
  address 10.34.37.1

auto e101-002-0
allow-hotplug e101-002-0
iface e101-002-0 inet static
  netmask 255.255.255.254
  address 10.35.37.1

auto e101-003-0
allow-hotplug e101-003-0
iface e101-003-0 inet static
  netmask 255.255.255.254
  address 10.36.37.1

auto e101-004-0
allow-hotplug e101-004-0
iface e101-004-0 inet static
  netmask 255.255.255.254
  address 10.136.137.1
```

* config - frr

```
opx-37# sh run
Building configuration...

Current configuration:
!
frr version 5.0.1
frr defaults traditional
hostname opx-37
log syslog informational
log facility local4
service integrated-vtysh-config
username cumulus nopassword
!
password zebra
!
interface e101-001-0
 description NAS## 0 25
!
interface e101-002-0
 description NAS## 0 29
!
interface e101-003-0
 description NAS## 0 33
!
interface e101-004-0
 description NAS## 0 37
!

...

!
interface lo
 ip address 10.96.0.37/32
!
router bgp 65037
 bgp router-id 10.96.0.37
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 bgp bestpath as-path multipath-relax
 neighbor 10.34.37.0 remote-as 65034
 neighbor 10.34.37.0 description veos-34
 neighbor 10.35.37.0 remote-as 65035
 neighbor 10.35.37.0 description sonic-35
 neighbor 10.36.37.0 remote-as 65036
 neighbor 10.36.37.0 description onie-36
 neighbor 10.136.137.0 remote-as 65036
 neighbor 10.136.137.0 description onie-36
 !
 address-family ipv4 unicast
  network 10.96.0.37/32
  neighbor 10.34.37.0 activate
  neighbor 10.34.37.0 soft-reconfiguration inbound
  neighbor 10.35.37.0 activate
  neighbor 10.35.37.0 soft-reconfiguration inbound
  neighbor 10.36.37.0 activate
  neighbor 10.36.37.0 soft-reconfiguration inbound
  neighbor 10.136.137.0 activate
  neighbor 10.136.137.0 soft-reconfiguration inbound
  maximum-paths 64
 exit-address-family
!
line vty
!
end
```


* bgp status

```
opx-37# show ip bgp summary

IPv4 Unicast Summary:
BGP router identifier 10.96.0.37, local AS number 65037 vrf-id 0
BGP table version 6
RIB entries 7, using 1064 bytes of memory
Peers 4, using 82 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
10.34.37.0      4      65034     123     120        0    0    0 01:52:30            3
10.35.37.0      4      65035     124     124        0    0    0 01:52:35            3
10.36.37.0      4      65036       0       0        0    0    0    never      Connect
10.136.137.0    4      65036       0       0        0    0    0    never      Connect

Total number of neighbors 4
opx-37#
```



### Cumulus

* config

```
cumulus@cumulus-38:~$ net show config command
net del all
net add time zone Etc/UTC
net add time ntp server 0.cumulusnetworks.pool.ntp.org iburst
net add time ntp server 1.cumulusnetworks.pool.ntp.org iburst
net add time ntp server 2.cumulusnetworks.pool.ntp.org iburst
net add time ntp server 3.cumulusnetworks.pool.ntp.org iburst
net add time ntp source eth0
net add snmp-server listening-address localhost
net add bgp autonomous-system 65038
net add routing defaults datacenter
net add routing service integrated-vtysh-config
net add routing log syslog informational
net add bgp neighbor 10.34.38.0 remote-as 65034
net add bgp neighbor 10.35.38.0 remote-as 65035
net add bgp neighbor 10.36.38.0 remote-as 65036
net add dns nameserver ipv4 192.168.1.1
net add ptp global slave-only no
net add ptp global priority1 255
net add ptp global priority2 255
net add ptp global domain-number 0
net add ptp global logging-level 5
net add ptp global path-trace-enabled no
net add ptp global use-syslog yes
net add ptp global verbose no
net add ptp global summary-interval 0
net add ptp global time-stamping
net add interface swp1 ip address 10.34.38.1/31
net add interface swp2 ip address 10.36.38.1/31
net add interface swp3 ip address 10.35.38.1/31
net add hostname cumulus-38
net add dot1x radius accounting-port 1813
net add dot1x radius authentication-port 1812
net add dot1x mab-activation-delay 30
net add dot1x eap-reauth-period 0
net commit
```

* bgp status

```
cumulus@cumulus-38:~$ net show bgp summary

show bgp ipv4 unicast summary
=============================
BGP router identifier 192.168.1.213, local AS number 65038 vrf-id 0
BGP table version 17
RIB entries 7, using 1064 bytes of memory
Peers 3, using 58 KiB of memory

Neighbor          V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
10.34.38.0        4      65034    2724    2723        0    0    0 02:15:10            4
sonic(10.35.38.0) 4      65035    2725    2723        0    0    0 02:15:10            4
10.36.38.0        4      65036    2643    2643        0    0    0 01:56:59            4

Total number of neighbors 3


show bgp ipv6 unicast summary
=============================
% No BGP neighbors found


show bgp l2vpn evpn summary
===========================
% No BGP neighbors found
```


### vEOS

* config

```
...

vrf definition MGMT
   rd 100:100
!
interface Ethernet1
   no switchport
   ip address 10.34.35.0/31
!
interface Ethernet2
   no switchport
   ip address 10.34.38.0/31
!
interface Ethernet3
   no switchport
   ip address 10.34.37.0/31
!
interface Ethernet4
   no switchport
   ip address 10.34.36.0/31
!

...

interface Loopback0
   ip address 10.96.0.34/32
!
interface Management1
   vrf forwarding MGMT
   ip address 192.168.1.34/24
!
ip route vrf MGMT 0.0.0.0/0 192.168.1.1
!
ip routing
no ip routing vrf MGMT
!
router bgp 65034
   router-id 10.96.0.34
   maximum-paths 32
   neighbor CORE peer-group
   neighbor CORE send-community
   neighbor CORE maximum-routes 100000
   neighbor 10.34.35.1 peer-group CORE
   neighbor 10.34.35.1 remote-as 65035
   neighbor 10.34.36.1 peer-group CORE
   neighbor 10.34.36.1 remote-as 65036
   neighbor 10.34.37.1 peer-group CORE
   neighbor 10.34.37.1 remote-as 65037
   neighbor 10.34.38.1 peer-group CORE
   neighbor 10.34.38.1 remote-as 65038
   !
   address-family ipv4
      neighbor CORE activate
      network 10.96.0.34/32
!
management api http-commands
   no shutdown
!
end
```

* bgp status

```
eos-34#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.96.0.34, local AS number 65034
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.34.35.1       4  65035            526       519    0    0 08:06:56 Estab   2      2
  10.34.36.1       4  65036            375       386    0    0 02:01:42 Estab   1      1
  10.34.37.1       4  65037            429       436    0    0 01:55:25 Estab   2      2
  10.34.38.1       4  65038           4095      4096    0    0 02:19:54 Estab   3      3
```



