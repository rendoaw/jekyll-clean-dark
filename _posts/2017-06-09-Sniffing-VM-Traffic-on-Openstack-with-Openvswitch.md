---
layout: post
comments: true
title: "Sniffing VM Traffic on Openstack with Openvswitch"
categories: blog
tags: 
  - openstack
  - networking
  - openvswitch
  - virtualization
date: 2017-06-09T22:39:55-04:00
---

This post is trying to illustrate how we can sniff the VM traffic from the hypervisor side. We assume that the openstack administrator does not have access to tenant VM to run tcpdump. 
For simplicity and familiarity, we will use the same setup and the same VM as we have in the previous post "Packet walk for Openstack with Openvswitch North South traffic".

Before we start the sniffer, let's define the plan:

* Sniff source
    * VM name: c-private
    * VM interface: eth0 (1st interface)

* Sniff destination
    * GRE to 192.168.1.142


Ideally, we should sniff the interface as close as possible to the VM. For Openstack with openvswitch, there are two possible locations:

* Option 1: TAP interface that directly connected to the VM
    * The best way to sniff the traffic because this TAP interface is directly connected to the VM.
    * Very easy to do it locally on the compute node itself. We can simply run tcpdump on TAP interface.
    * It will be quite tricky if we want to mirror the packet to remote destination. In this case we need to use "tc" utility and combine it with a tunnel.
    > Mirroring to remote destination is out of scope this post


* Option 2: qvo port, the closest ovs interface to the VM
    * Please remember that this interface is not directly connected to the VM because there is a linux bridge that sit between VM and Openvswitch. This linux bridge performs any VM specific filtering as well as QoS implementation. 
    * So, in theory, mirroring on this port is OK if we focus more on the traffic sent by the VM. For traffic that is going to the VM, there is a chance that the sniffer is able to see the packet but the VM never receive it due to blocked by the iptables inside linux bridge.
    * Mirroring packet on openvswitch port is relatively easy and we can also send the mirrored packet to the remote destination via GRE tunnel.



## Basic info gathering

Before we can sniff the packet, we need to know which compute node that this VM is located, and what is the TAP interface that associated with it. For Openvswitch, we also need to know which qvo port is facing this VM.

* Basic VM info

    ```
    # nova show c-private
    +--------------------------------------+----------------------------------------------------------------------------------+
    | Property                             | Value                                                                            |
    +--------------------------------------+----------------------------------------------------------------------------------+
    | OS-DCF:diskConfig                    | AUTO                                                                             |
    | OS-EXT-AZ:availability_zone          | nova                                                                             |
    | OS-EXT-SRV-ATTR:host                 | compute01                                                                        |
    | OS-EXT-SRV-ATTR:hostname             | c-private                                                                        |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | compute01                                                                        |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000003                                                                |
    | OS-EXT-SRV-ATTR:kernel_id            |                                                                                  |
    | OS-EXT-SRV-ATTR:launch_index         | 0                                                                                |
    | OS-EXT-SRV-ATTR:ramdisk_id           |                                                                                  |
    | OS-EXT-SRV-ATTR:reservation_id       | r-vmegcw9h                                                                       |
    | OS-EXT-SRV-ATTR:root_device_name     | /dev/vda                                                                         |
    | OS-EXT-SRV-ATTR:user_data            | -                                                                                |
    | OS-EXT-STS:power_state               | 1                                                                                |
    | OS-EXT-STS:task_state                | -                                                                                |
    | OS-EXT-STS:vm_state                  | active                                                                           |
    | OS-SRV-USG:launched_at               | 2017-05-31T06:52:36.000000                                                       |
    | OS-SRV-USG:terminated_at             | -                                                                                |
    | accessIPv4                           |                                                                                  |
    | accessIPv6                           |                                                                                  |
    | config_drive                         |                                                                                  |
    | created                              | 2017-05-31T06:52:17Z                                                             |
    | description                          | c-private                                                                        |
    | flavor                               | m1.tiny (1)                                                                      |
    | hostId                               | ec6b2b705a4dfd3910541ca83850deedb3f9ac134e21eee9e5f6f048                         |
    | host_status                          | UP                                                                               |
    | id                                   | ced81387-f223-4e1d-9068-69b3c20fe94f                                             |
    | image                                | Attempt to boot from volume - no image supplied                                  |
    | key_name                             | -                                                                                |
    | locked                               | False                                                                            |
    | metadata                             | {}                                                                               |
    | name                                 | c-private                                                                        |
    | net1 network                         | 172.19.1.3, 172.24.4.3                                                           |
    | os-extended-volumes:volumes_attached | [{"id": "6ecb2dfb-92df-45b1-8bfb-c00ff5e7708f", "delete_on_termination": false}] |
    | progress                             | 0                                                                                |
    | security_groups                      | default                                                                          |
    | status                               | ACTIVE                                                                           |
    | tags                                 | []                                                                               |
    | tenant_id                            | 0283565b977d4bdaaa57fa8bdf2e0159                                                 |
    | updated                              | 2017-06-06T17:25:48Z                                                             |
    | user_id                              | 8ee62c67fdf44a028c05776ab6cc8218                                                 |
    +--------------------------------------+----------------------------------------------------------------------------------+
    ```

* OK, now we have the fllowing info

    * VM instance name: instance-00000003
    * Hypervisor: compute01
    * network: net1
    * Fixed IP address: 172.19.1.3
    * Floating IP: 172.24.4.3


* OK, now we go to compute node (compute01) and find the TAP interface

```
compute01# virsh dumpxml instance-00000003 | egrep "mac|tap"                                                                
      <mac address='fa:16:3e:3b:53:26'/>
      <target dev='tapf7eae624-34'/>
```


* OK, we know the tap interface = *tapf7eae624-34*. We can use the same method as the previous post where we go hop-by-hop to find the actual path, or we an simply follow the convention. 

* Most of the time, openvswitch qvo interface name has similar name as the TAP interface. For example, in this case, the qvo port name is *qvof7eae624-34*



## Scenario 1: tcpudmp locally on TAP interface

For proof of concept, let's run continuos ping while we try to sniff the packets

```
# ping 172.24.4.3
```


The following shows tcpdump that running on hypervisor itself

```
# tcpdump -n -i tapf7eae624-34
tcpdump: WARNING: tapf7eae624-34: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tapf7eae624-34, link-type EN10MB (Ethernet), capture size 65535 bytes
22:23:09.549480 IP 172.24.4.1 > 172.19.1.3: ICMP echo request, id 14347, seq 23, length 64
22:23:09.549624 IP 172.19.1.3 > 172.24.4.1: ICMP echo reply, id 14347, seq 23, length 64
22:23:10.550865 IP 172.24.4.1 > 172.19.1.3: ICMP echo request, id 14347, seq 24, length 64
22:23:10.551037 IP 172.19.1.3 > 172.24.4.1: ICMP echo reply, id 14347, seq 24, length 64
22:23:11.549847 ARP, Request who-has 172.19.1.1 tell 172.19.1.3, length 28
22:23:11.551562 ARP, Reply 172.19.1.1 is-at fa:16:3e:ca:72:d8, length 28
22:23:11.551920 IP 172.24.4.1 > 172.19.1.3: ICMP echo request, id 14347, seq 25, length 64
22:23:11.552005 IP 172.19.1.3 > 172.24.4.1: ICMP echo reply, id 14347, seq 25, length 64
...
```


## Scenario 2: mirror packet and send to remote destination

Same as scenario 1, let's run continuos ping while we try to sniff the packets

```
# ping 172.24.4.3
```

The following shows how to mirror and send the mirrored packet to remote destination

```
ovs-vsctl add-port br-int gre0 \
    -- set interface gre0 type=gre options:remote_ip=192.168.1.142 \
    -- --id=@p get port gre0 \
    -- --id=@target get Port qvof7eae624-34 \
    -- --id=@m create mirror name=m0 output-port=@p select-dst-port=@target select-src-port=@target \
    -- set bridge br-int mirrors=@m
```

* a quick explanation about the command above

    * we create a new ovs port called gre0 and attach this interface to br-int bridge
        * we choose br-int bridge because this is the bridge that port *qvof7eae624-34* is connected.

    * this gre0 interface is actually a GRE port with destination = 192.168.1.142.
        * this is the place where we will run tcpdump/wireshark to decode the packets.

    * get the reference to this newly created gre0 port and save to variable/pointer "@p"
    
    * get the reference to the port that we want to mirror, which is *qvof7eae624-34* and save it to variable "@target"

    * define the mirroring rule
        * name: m0
        * mirror incoming port: "@target" which is port *qvof7eae624-34*
        * mirror outgoing port: "@target" which is port *qvof7eae624-34*
        > this means we mirror both ingress and egress packets on *qvof7eae624-34*
    
    * enable the mirroring on br-int bridge


* And here is the result

    ```
    remote-node-192-168.1.152# tcpdump -n -i en0 proto 47
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
    21:41:48.569344 IP 192.168.1.13 > 192.168.1.142: GREv0, length 106: IP 172.24.4.1 > 172.19.1.3: ICMP echo request, id 12593, seq 1, length 64
    21:41:48.569611 IP 192.168.1.13 > 192.168.1.142: GREv0, length 106: IP 172.19.1.3 > 172.24.4.1: ICMP echo reply, id 12593, seq 1, length 64
    21:41:49.664341 IP 192.168.1.13 > 192.168.1.142: GREv0, length 106: IP 172.24.4.1 > 172.19.1.3: ICMP echo request, id 12593, seq 2, length 64
    21:41:49.664347 IP 192.168.1.13 > 192.168.1.142: GREv0, length 106: IP 172.19.1.3 > 172.24.4.1: ICMP echo reply, id 12593, seq 2, length 64
    21:41:53.604873 IP 192.168.1.13 > 192.168.1.142: GREv0, length 50: ARP, Request who-has 172.19.1.1 tell 172.19.1.3, length 28
    21:41:53.604888 IP 192.168.1.13 > 192.168.1.142: GREv0, length 50: ARP, Reply 172.19.1.1 is-at fa:16:3e:ca:72:d8, length 28
    ...
    ```
    
* From the tcpdump output above we can see that both ingress and egress packets on *qvof7eae624-34* are sent to remote-capture node with GRE encapsulation. 


* Misc

    * to verify if mirroring is enable on openvswitch, we can do

        ```
        # ovs-vsctl list Bridge br-int
        _uuid               : 354c2a5f-266c-4d97-a8c4-2ed4877b47ab
        auto_attach         : []
        controller          : [06d57159-679a-4a44-9d53-3316f77581f3]
        datapath_id         : "00005e2a4c35974d"
        datapath_type       : system
        datapath_version    : "<unknown>"
        external_ids        : {}
        fail_mode           : secure
        flood_vlans         : []
        flow_tables         : {}
        ipfix               : []
        mcast_snooping_enable: false
        mirrors             : [c67d76af-bdca-4391-a494-cb094bb7c665]   -----> mirroring in enabled.
        name                : br-int
        netflow             : []
        other_config        : {}
        ports               : [49404b86-eee9-449c-b557-d1f7712d8dd3, 5da9153d-fc5d-4533-93ea-d8acb98898a9, 615f38fd-3f4f-4ddb-b42a-e2be78a2db3f, 6fcba884-b8a7-4d20-8ca6-57bef99b1796, 8117274f-5b52-4792-9564-9d0c691aa58a]
        protocols           : ["OpenFlow10", "OpenFlow13"]
        rstp_enable         : false
        rstp_status         : {}
        sflow               : []
        status              : {}
        stp_enable          : false
        ```

    * to completely disable/remove the mirror

        ```
        # ovs-vsctl clear bridge br-int mirrors
        # ovs-vsctl del-port br-int gre0
        ```

