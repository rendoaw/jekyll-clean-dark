---
layout: post
comments: true
title: "Packet walk for Openstack with Openvswitch North South traffic"
categories: blog
tags: 
  - openstack
  - networking
  - openvswitch
  - virtualization
date: 2017-06-09T15:39:55-04:00
---

This post will try to illustrate how the Openstack VM packet sent from the VM up until it reach the external network.  In this exercise, we are going to use vanilla Openstack with Openvswitch as neutron plugin.  
We will use the following diagram from (https://docs.openstack.org/liberty/networking-guide/scenario-classic-ovs.html) as our reference.

* Compute node network components
    ![Compute node network components](https://docs.openstack.org/liberty/networking-guide/_images/scenario-classic-ovs-compute2.png)

* Network node network components
    ![Network node network components](https://docs.openstack.org/liberty/networking-guide/_images/scenario-classic-ovs-network2.png)



## Basic info gathering

First thing first, let's collect some information about our Lab setup.  For simplicity, let focus on a single VM and then we will try to find out how the connection is build.
So, Let's pick a VM name "c-private", and for now we are going to focus on north-south traffic.


* First, let's find which compute that host this VM

    ```
    controller# nova show c-private | egrep "instance|hypervis|name"
    | OS-EXT-SRV-ATTR:hostname             | c-private                                                                        |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | compute01                                                                        |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000003                                                                |
    | OS-EXT-SRV-ATTR:root_device_name     | /dev/vda                                                                         |
    | name                                 | c-private                                                                        |
    ```


* Then, find the IP and MAC address of this VM

    ```
    controller# nova interface-list c-private
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+ 
    | Port State | Port ID                              | Net ID                               | IP addresses | MAC Addr          | 
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+ 
    | ACTIVE     | f7eae624-3488-4ccb-962a-d94f9b86aeb7 | 1763ddbb-7689-4266-919d-e250e7577749 | 172.19.1.3   | fa:16:3e:3b:53:26 | 
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+ 
    ```


* Summary

    * Here is what we know so far
    
        * VM name: c-private
        * YM IP: 172.19.1.3
        * VM mac address: fa:16:3e:3b:53:26
        * compute node: compute01
        * instance name: instance-00000003 



Now we are going to trace the path of the packet from the VM itself until the packet reach the external network.


## Packet walk @ Compute Node

Let's start from the VM. We need to go to the compute node by ssh into it

* First, we know that VM is connected to the host via TAP interface. This TAP interface will be used to send the outgoing traffic from the VM. So, we need to find which TAP interface is connected to this VM

* To find the tap interface associated to this VM NIC, the only way that i know is by using classic Libvirt virsh command on that particular VM instance. From the previous step, we know that the VM instance id is "instance-00000003"

    ```
    compute01# virsh dumpxml instance-00000003 | egrep "mac|tap"                                                                
          <mac address='fa:16:3e:3b:53:26'/>
          <target dev='tapf7eae624-34'/>
    ```

* OK, now we know that VM "c-private" is connected via *tapf7eae624-34*.

* According to the compute node network diagram in the beginning of this post, this TAP interface should be connected to a linux bride first for firewall policy and/or QoS policy implementation before actually connected to Openvswitch.

* Let's check linux bridge table on compute node

    ```
    compute-node# brctl show                                                                                                       
    bridge name     bridge id               STP enabled     interfaces                                                              
    qbr1d939c00-e4          8000.762df08bd21e       no              qvb1d939c00-e4                                                  
                                                            tap1d939c00-e4                                                          
    qbrf7eae624-34          8000.c6fbe09bd3ee       no              qvbf7eae624-34                                                  
                                                            tapf7eae624-34        			
    ```

* Great! The above output shows "tapf7eae624-34" is connected to a linux bridge name *qbrf7eae624-34* and this *qbrf7eae624-34* bridge has another virtual interface named *qvbf7eae624-34*.
    
* So, to review, our packet flow now become

    ```
    c-private VM vNIC -- tapf7eae624-34 -- bridge qbrf7eae624-34 -- qvbf7eae624-34 
    ```


* Based on the documentation again, the next component would be the Openvswitch itself. 

* Now, we check openvswitch configuration and check how this tap interface is connected 

    ```
    compute01# ovs-vsctl show

    9b36fd19-21cb-4d25-b590-1f822c18d373                                                                                            
        Manager "ptcp:6640:127.0.0.1"                                                                                               
            is_connected: true                                                                                                      
        Bridge br-tun                                                                                                               
            Controller "tcp:127.0.0.1:6633"                                                                                         
                is_connected: true                                                                                                  
            fail_mode: secure                                                                                                       
            Port "vxlan-c0a8010c"                                                                                                   
                Interface "vxlan-c0a8010c"                                                                                          
                    type: vxlan                                                                                                     
                    options: {df_default="true", in_key=flow, local_ip="192.168.1.13", out_key=flow, remote_ip="192.168.1.12"}      
            Port br-tun                                                                                                             
                Interface br-tun                                                                                                    
                    type: internal                                                                                                  
            Port patch-int                                                                                                          
                Interface patch-int                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=patch-tun}               
        Bridge br-int                                                                                                               
            Controller "tcp:127.0.0.1:6633"                                                                                         
                is_connected: true                                                                                                  
            fail_mode: secure                                                                                                       
            Port "qvo1d939c00-e4"                                                                                                   
                tag: 1                                                                                                              
                Interface "qvo1d939c00-e4"                                                                                          
            Port br-int                                                                                                             
                Interface br-int                                                                                                    
                    type: internal                                                                                                  
            Port patch-tun                                                                                                          
                Interface patch-tun                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=patch-int}                                                                                       
            Port "qvof7eae624-34"                                                                                                   
                tag: 2
                Interface "qvof7eae624-34" 
                
    ```

* What can we get from above output

    * virtual interface *qvof7eae624-34* is connected to openvswitch br-int bridge and have internal tag=2 assigned by OVS.
    * after that, the packet will be forwarded to br-tun bridge via patch interface.
    * This means, untagged packet coming from the VM is received via **qvof7eae624-34* and then sent out to to br-tun bridge with additional internal tag.


* According to the Compute node network components diagram, for north-south traffic, br-tun will forward the packet to network node using vxlan tunnel. 

* Unfortunately, the output of "ovs-vsctl show" does not tell us about the mapping between internal tag and the vxlan VNI ID. 

* Fortunately, i got answer from [ask.openstack.org](https://ask.openstack.org/en/question/107366/how-to-find-the-mapping-between-vxlan-vni-and-openvswitch-internal-tag) on how to find the mapping. 

* So, let's check the flow table inside openvswitch on compute node. For this, we need to know the VNI ID (segmentation ID) that assigned by Openstack for the virtual network that the VM is connected.

* Here is the virtual network where VM "c-private" is connected

    ```
    controller# nova show c-private | grep network
    | net1 network                         | 172.19.1.3, 172.24.4.3                                                           |
    ```

* OK, we know the VM is connected to *net1*. Next step, find out what is the vxlan VNI ID for this virtual network

    ```
    controller# neutron net-show net1                                                 
    +---------------------------+--------------------------------------+                                         
    | Field                     | Value                                |                                         
    +---------------------------+--------------------------------------+                                         
    | admin_state_up            | True                                 |                                         
    | availability_zone_hints   |                                      |                                         
    | availability_zones        | nova                                 |                                         
    | created_at                | 2017-05-30T01:14:03Z                 |                                         
    | description               |                                      |                                         
    | id                        | 1763ddbb-7689-4266-919d-e250e7577749 |                                         
    | ipv4_address_scope        |                                      |                                         
    | ipv6_address_scope        |                                      |                                         
    | mtu                       | 1450                                 |                                         
    | name                      | net1                                 |                                         
    | project_id                | 0283565b977d4bdaaa57fa8bdf2e0159     |                                         
    | provider:network_type     | vxlan                                |                                         
    | provider:physical_network |                                      |                                         
    | provider:segmentation_id  | 90                                   |                                         
    | revision_number           | 11                                   |                                         
    | router:external           | False                                |                                         
    | shared                    | True                                 |                                         
    | status                    | ACTIVE                               |                                         
    | subnets                   | 0685edba-9a91-4bc8-badc-5427e716693a |                                         
    | tags                      |                                      |                                         
    | tenant_id                 | 0283565b977d4bdaaa57fa8bdf2e0159     |                                         
    | updated_at                | 2017-05-31T06:44:28Z                 |                                         
    +---------------------------+--------------------------------------+                  
    ```

* From the output above, we know that the vxlan ID is 90 decimal (0x5a in hexadescimal)

* Let's also find out the MAC address of VM 'c-private' default gateway, by searching neutron ports that belong to the net1 subnet "0685edba-9a91-4bc8-badc-5427e716693a" above.

    ```
    controller# neutron port-list | grep 0685edba-9a91-4bc8-badc-5427e716693a
    | 2c628d36-765c-420b-b008-b6311a0ed17a |      | 0283565b977d4bdaaa57fa8bdf2e0159 | fa:16:3e:ca:72:d8 | {"subnet_id": "0685edba-9a91-4bc8-badc-5427e716693a", "ip_address": "172.19.1.1"}  |
    | 3775bfe7-6ed2-4a95-883e-701a6d04e6a2 |      | 0283565b977d4bdaaa57fa8bdf2e0159 | fa:16:3e:65:8c:df | {"subnet_id": "0685edba-9a91-4bc8-badc-5427e716693a", "ip_address": "172.19.1.2"}  |
    | f7eae624-3488-4ccb-962a-d94f9b86aeb7 |      | 0283565b977d4bdaaa57fa8bdf2e0159 | fa:16:3e:3b:53:26 | {"subnet_id": "0685edba-9a91-4bc8-badc-5427e716693a", "ip_address": "172.19.1.3"}  |
    ```

* Before checking the ovs table, we should generate some traffic to make sure ovs has entries on it

    ```
    c-private$ ping 172.19.1.1
    PING 172.19.1.1 (172.19.1.1): 56 data bytes
    64 bytes from 172.19.1.1: seq=0 ttl=64 time=1.254 ms
    64 bytes from 172.19.1.1: seq=1 ttl=64 time=0.847 ms
    64 bytes from 172.19.1.1: seq=2 ttl=64 time=1.539 ms
    
    --- 172.19.1.1 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.847/1.213/1.539 ms
    ```


* now, let's also dump the mac address table on the compute node ovs

    ```
    compute01# ovs-appctl fdb/show br-tun
    port  VLAN  MAC                Age

    compute01# ovs-appctl fdb/show br-int
    port  VLAN  MAC                Age
        2     2  fa:16:3e:3b:53:26    0    --> mac address of VM c-private
        1     2  fa:16:3e:ca:72:d8    0    --> mac address of net1 default gateway
    ```


* And also list the ovs port

    ```
    compute01# ovs-ofctl show br-tun
    ...
    1(patch-int): addr:8e:ac:86:71:04:ce
        config:     0
        state:      0
        speed: 0 Mbps now, 0 Mbps max
    2(vxlan-c0a8010c): addr:ba:d7:23:6c:d0:06
        config:     0
        state:      0
        speed: 0 Mbps now, 0 Mbps max
    LOCAL(br-tun): addr:7a:d1:cc:0f:f3:45
        config:     PORT_DOWN
        state:      LINK_DOWN
        speed: 0 Mbps now, 0 Mbps max
    OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0


    compute01# ovs-ofctl show br-int
    ...
    1(patch-tun): addr:66:b0:4e:2c:a7:a5
        config:     0
        state:      0
        speed: 0 Mbps now, 0 Mbps max
    2(qvof7eae624-34): addr:d6:bb:95:ae:c9:a3
        config:     0
        state:      0
        current:    10GB-FD COPPER
        speed: 10000 Mbps now, 0 Mbps max
    3(qvo1d939c00-e4): addr:f6:7a:d2:4b:a0:ea
        config:     0
        state:      0
        current:    10GB-FD COPPER
        speed: 10000 Mbps now, 0 Mbps max
    LOCAL(br-int): addr:5e:2a:4c:35:97:4d
        config:     PORT_DOWN
        state:      LINK_DOWN
        speed: 0 Mbps now, 0 Mbps max
    OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
    ```


* From the port list and mac address table output above we have information about:
    * *qvof7eae624-34* is OVS bridge br-int port 2
    * vxlan tunnel to network node *vxlan-c0a8010c* is OVS bridge br-tun port 2

* Ok now we go to the flow table

    ```
    compute01# ovs-ofctl dump-flows br-int
    ...
    cookie=0xb950e065fe76cc7b, duration=114016.867s, table=0, n_packets=142979, n_bytes=14018035, idle_age=0, hard_age=65534, priority=9,in_port=2 actions=resubmit(,25)
    
    cookie=0xb950e065fe76cc7b, duration=114016.870s, table=25, n_packets=153695, n_bytes=14468051, idle_age=0, hard_age=65534, priority=2,in_port=2,dl_src=fa:16:3e:3b:53:26 actions=NORMAL
    ...
   


    compute01# ovs-ofctl dump-flows br-tun
    ...
    cookie=0x876248e21ebb5890, duration=345233.029s, table=0, n_packets=155284, n_bytes=14620597, idle_age=0, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
    
    cookie=0x876248e21ebb5890, duration=345233.026s, table=2, n_packets=155246, n_bytes=14617083, idle_age=0, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)

    cookie=0x876248e21ebb5890, duration=142911.162s, table=20, n_packets=153436, n_bytes=14442734, hard_timeout=300, idle_age=0, hard_age=0, priority=1,vlan_tci=0x0002/0x0fff,dl_dst=fa:16:3e:ca:72:d8 actions=load:0->NXM_OF_VLAN_TCI[],load:0x5a->NXM_NX_TUN_ID[],output:2

    cookie=0x876248e21ebb5890, duration=14.744s, table=20, n_packets=4, n_bytes=336, hard_timeout=300, idle_age=9, hard_age=9, priority=1,vlan_tci=0x0002/0x0fff,dl_dst=fa:16:3e:65:8c:df actions=load:0->NXM_OF_VLAN_TCI[],load:0x5a->NXM_NX_TUN_ID[],output:2

    cookie=0x876248e21ebb5890, duration=222938.295s, table=22, n_packets=17, n_bytes=1528, idle_age=14, hard_age=65534, priority=1,dl_vlan=2 actions=strip_vlan,load:0x5a->NXM_NX_TUN_ID[],output:2
    ...
    ```

* From the flow table above we can see that
    * Packet arrived from the VM on OVS bridge br-int port 2 on table=0 will be redirected to table=25. Additionally, from the config we also know that all packet arrived in br-int port 2 will be assigned internal tag = 2
    * In table=25, there is a rule that match with VM source mac address. This rule has action=NORMAL, which mean OVS will do normal mac address lookup.
    * Based on mac address table, packet that sent from the VM to default gateway 172.19.1.1/fa:16:3e:ca:72:d8 will be forwarded to br-int port 1 which is patch interface to br-tun.
    * Inside br-tun bridge table=0, packet that coming from br-int patch interface, which is ovs br-tun port 1, will be redirected to table 2
    * Then, the packet will hit another rule in table=2 where ovs will redirect the packet again to table=20
    * In table=20 the packet will hit one of the rule. 
        * For this example, we focus on the packet going to default gateway 172.19.1.1/fa:16:3e:ca:72:d8
    * OVS will then remove any internal ovs tag (load:0->NXM_OF_VLAN_TCI[]) and add vxlan VNI id = 0x5a (90 in decimal)
    * Packet sent to network node inside vxlan tunnel



* So, to review, our packet flow now become

    ```
    c-private VM vNIC -- tapf7eae624-34 -- bridge qbrf7eae624-34 -- qvbf7eae624-34 -- ovs br-int -- qvof7eae624-34 -- patch-tun -- ovs br-tun -- patch-int -- vxlan vni 90
    ```




## Packet walk @ network node


At the end of the previous step, the packet was sent to network node inside vxlan tunnel. This section will cover the packet walk on the network node. For clarity, i remove some non-relevant parts.


* Let's check the ovs config first

    ```
    7b14d048-c73e-4c8a-af43-cf568cf79d6d
        Manager "ptcp:6640:127.0.0.1"                                                                                               
            is_connected: true                                                                                                      
        Bridge br-ex                                                                                                                
            Controller "tcp:127.0.0.1:6633"                                                                                         
                is_connected: true                                                                                                  
            fail_mode: secure                                                                                                       
            Port br-ex                                                                                                              
                Interface br-ex                                                                                                     
                    type: internal                                                                                                  
            Port phy-br-ex                                                                                                          
                Interface phy-br-ex                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=int-br-ex}                                                                                       
        Bridge br-int                                                                                                               
            Controller "tcp:127.0.0.1:6633"                                                                                         
                is_connected: true                                                                                                  
            fail_mode: secure                                                                                                       
            Port "tap2b56c518-fb"                                                                                                   
                tag: 1                                                                                                              
                Interface "tap2b56c518-fb"                                                                                          
                    type: internal                                                                                                  
            Port "tap3775bfe7-6e"                                                                                                   
                tag: 3                                                                                                              
                Interface "tap3775bfe7-6e"                                                                                          
                    type: internal                                                                                                  
            Port "qg-421b407a-81"                                                                                                   
                tag: 4                                                                                                              
                Interface "qg-421b407a-81"                                                                                          
                    type: internal                                                                                                  
            Port "qg-ede3b53d-da"                                                                                                   
                tag: 4                                                                                                              
                Interface "qg-ede3b53d-da"                     
                    type: internal                                                                                                  
            Port int-br-ex                                                                                                          
                Interface int-br-ex                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=phy-br-ex}                                                                                       
            Port "qg-a1b3323f-35"                                                                                                   
                tag: 4                                                                                                              
                Interface "qg-a1b3323f-35"                                                                                          
                    type: internal                                                                                                  
            Port "qr-2c628d36-76"                                                                                                   
                tag: 3                                                                                                              
                Interface "qr-2c628d36-76"                                                                                          
                    type: internal                                                                                                  
            Port "qr-b9a4d300-68"                                                                                                   
                tag: 2                                                                                                              
                Interface "qr-b9a4d300-68"                                                                                          
                    type: internal                                                                                                  
            Port "tap80ac2f6b-87"                                                                                                   
                tag: 2                                                                                                              
                Interface "tap80ac2f6b-87"                                                                                          
                    type: internal                                                                                                  
            Port br-int                                                                                                             
                Interface br-int                                                                                                    
                    type: internal                                                                                                  
            Port "qr-f80a6a75-e6"                                                                                                   
                tag: 1                                                                                                              
                Interface "qr-f80a6a75-e6"                                                                                          
                    type: internal                                                                                                  
            Port patch-tun                                                                                                          
                Interface patch-tun                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=patch-int}
        Bridge br-tun
            Controller "tcp:127.0.0.1:6633"
                is_connected: true                                                                                                  
            fail_mode: secure                                                                                                       
            Port br-tun                                                                                                             
                Interface br-tun                                                                                                    
                    type: internal                                                                                                  
            Port "vxlan-c0a8010d"                                                                                                   
                Interface "vxlan-c0a8010d"                                                                                          
                    type: vxlan                                                                                                     
                    options: {df_default="true", in_key=flow, local_ip="192.168.1.12", out_key=flow, remote_ip="192.168.1.13"}      
            Port patch-int                                                                                                          
                Interface patch-int                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=patch-tun}                                                                                       
        ovs_version: "2.6.1"                        
    ```
	  

* Same as before, ovs config doesn't tell us the mapping between the vxlan id and internal tag. So we need to check the flow table again in network node

    * mac-address table

        ```
        network-node# ovs-appctl fdb/show br-tun
         port  VLAN  MAC                Age
        
        network-node# ovs-appctl fdb/show br-int
         port  VLAN  MAC                Age
            3     3  fa:16:3e:3b:53:26    1
            8     3  fa:16:3e:ca:72:d8    1
            1     4  ae:05:ca:a5:1e:4d    1
           11     4  fa:16:3e:94:5d:70    1
        ```


    * br-tun port list

        ```
        network-node# ovs-ofctl show br-tun
         ...
         1(patch-int): addr:c6:ca:7a:8a:b7:09
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         2(vxlan-c0a8010d): addr:42:ae:83:de:45:87
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         LOCAL(br-tun): addr:2a:40:69:cb:0b:4a
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
        ...
        ```

    * br-int port list

        ```
        network-node# ovs-ofctl show br-int
         ...
         1(int-br-ex): addr:de:88:5e:32:fc:16
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         2(tap80ac2f6b-87): addr:00:00:00:00:02:00
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         3(patch-tun): addr:fe:97:fd:ad:dd:17
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         4(tap2b56c518-fb): addr:00:00:00:00:20:9d
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         5(tap3775bfe7-6e): addr:00:00:00:00:00:0a
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         6(qr-b9a4d300-68): addr:00:00:00:00:f0:c6
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         7(qr-f80a6a75-e6): addr:00:00:00:00:02:00
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         8(qr-2c628d36-76): addr:00:00:00:00:c0:b3
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         9(qg-421b407a-81): addr:00:00:00:00:90:78
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         10(qg-ede3b53d-da): addr:00:00:00:00:b0:fb
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         11(qg-a1b3323f-35): addr:00:00:00:00:60:ca
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         LOCAL(br-int): addr:ce:89:0c:a6:87:4d
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
        ...
        ```


    * br-tun flow table

        ```
        network-node# ovs-ofctl dump-flows br-tun
         ...
         cookie=0xb5b1b0a3b000ddbe, duration=223464.064s, table=0, n_packets=155744, n_bytes=14660419, idle_age=0, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)

         cookie=0xb5b1b0a3b000ddbe, duration=223376.366s, table=4, n_packets=154264, n_bytes=14518735, idle_age=0, hard_age=65534, priority=1,tun_id=0x5a actions=mod_vlan_vid:3,resubmit(,10)
         
         cookie=0xb5b1b0a3b000ddbe, duration=223465.975s, table=10, n_packets=155744, n_bytes=14660419, idle_age=0, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0xb5b1b0a3b000ddbe,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:OXM_OF_IN_PORT[]),output:1
         
        ```

    * br-int flow table

        ```
        network-node# ovs-ofctl dump-flows br-int
         ...
         cookie=0x86954af3f33dce38, duration=223464.386s, table=0, n_packets=463806, n_bytes=43847428, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
         ...
        ```


* So, what do we get here:

    * Packet from compute node is received on br-tun port *vxlan-c0a8010d*
    * *vxlan-c0a8010d* has port id = 2
    * from br-tun flow table=0, any packet received on port 2, will be redirected to table 4
    * In table 4, packet received with vxlan vni id 0x5a (90) will be redirected to table 10 and add internal vlan tag = 3
    * In table 10, send all packet to port 1
    * br-tun port 1 is patch interface to br-int, so now the packet is in br-int bridge
    * Inside br-int table 0, it will hit generic rule with action NORMAL which mean ovs will find the outgoing port based on mac address table
    * From br-int mac address table we see that default gateway 172.19.1.1/fa:16:3e:ca:72:d8 is in br-int port 8 which is *qr-2c628d36-76*
    * According the documentation, this *qr-2c628d36-76* will be connected a router namespace


* There could be multiple neutron router instances

    ```
    # neutron router-list
    +------------------------+---------+------------------------+-------------------------+-------------+-------+
    | id                     | name    | tenant_id              | external_gateway_info   | distributed | ha    |
    +------------------------+---------+------------------------+-------------------------+-------------+-------+
    | 0b0cb65f-3739-4ca8     | r2      | 0283565b977d4bdaaa57fa | {"network_id": "d478a8b | False       | False |
    | -a68b-271545681aa7     |         | 8bdf2e0159             | 8-96fa-4996-a472-938215 |             |       |
    |                        |         |                        | b0d1e2", "enable_snat": |             |       |
    |                        |         |                        | false,                  |             |       |
    |                        |         |                        | "external_fixed_ips":   |             |       |
    |                        |         |                        | [{"subnet_id":          |             |       |
    |                        |         |                        | "b193dc42-77ac-44cb-862 |             |       |
    |                        |         |                        | 2-f15331cf6319",        |             |       |
    |                        |         |                        | "ip_address":           |             |       |
    |                        |         |                        | "172.24.4.2"}]}         |             |       |
    | 58731090-1c82-4aa5-b31 | router1 | d67ee95c357642549d48c2 | {"network_id": "d478a8b | False       | False |
    | 9-1598c7e6ae06         |         | ea36250947             | 8-96fa-4996-a472-938215 |             |       |
    |                        |         |                        | b0d1e2", "enable_snat": |             |       |
    |                        |         |                        | true,                   |             |       |
    |                        |         |                        | "external_fixed_ips":   |             |       |
    |                        |         |                        | [{"subnet_id":          |             |       |
    |                        |         |                        | "b193dc42-77ac-44cb-862 |             |       |
    |                        |         |                        | 2-f15331cf6319",        |             |       |
    |                        |         |                        | "ip_address":           |             |       |
    |                        |         |                        | "172.24.4.9"}]}         |             |       |
    | bfc87e1e-6b93-446f-a30 | r1      | 0283565b977d4bdaaa57fa | {"network_id": "d478a8b | False       | False |
    | 5-6dc6153c1198         |         | 8bdf2e0159             | 8-96fa-4996-a472-938215 |             |       |
    |                        |         |                        | b0d1e2", "enable_snat": |             |       |
    |                        |         |                        | true,                   |             |       |
    |                        |         |                        | "external_fixed_ips":   |             |       |
    |                        |         |                        | [{"subnet_id":          |             |       |
    |                        |         |                        | "b193dc42-77ac-44cb-862 |             |       |
    |                        |         |                        | 2-f15331cf6319",        |             |       |
    |                        |         |                        | "ip_address":           |             |       |
    |                        |         |                        | "172.24.4.12"}]}        |             |       |
    +------------------------+---------+------------------------+-------------------------+-------------+-------+
    ```

* And, each router instance is associated to a single network namespace

    ```
    network-node # ip netns list                                                                            
    qrouter-0b0cb65f-3739-4ca8-a68b-271545681aa7                                                                                    
    qrouter-bfc87e1e-6b93-446f-a305-6dc6153c1198                                                                                    
    qrouter-58731090-1c82-4aa5-b319-1598c7e6ae06                                                                                    
    qdhcp-c218c762-e820-464b-b5b9-cfeb9eb51f89                                                                                      
    qdhcp-1763ddbb-7689-4266-919d-e250e7577749                                                                                      
    qdhcp-ab481d2f-05ac-4dea-a37d-0e0c8ab38444
    ```

* Unfortunately, the list output above does not contain any information about the internal network id or internal subnet, so we have to find it by doing neutron router-port-list command on each router instance. For clarity i will only show the correct router instance below.

    ```
    # neutron router-port-list bfc87e1e-6b93-446f-a305-6dc6153c1198         
    +--------------------------+------+--------------------------+-------------------+--------------------------+
    | id                       | name | tenant_id                | mac_address       | fixed_ips                |
    +--------------------------+------+--------------------------+-------------------+--------------------------+
    | 2c628d36-765c-           |      | 0283565b977d4bdaaa57fa8b | fa:16:3e:ca:72:d8 | {"subnet_id": "0685edba- |
    | 420b-b008-b6311a0ed17a   |      | df2e0159                 |                   | 9a91-4bc8-badc-          |
    |                          |      |                          |                   | 5427e716693a",           |
    |                          |      |                          |                   | "ip_address":            |
    |                          |      |                          |                   | "172.19.1.1"}            |
    | a1b3323f-35c5-44b8-9e6d- |      |                          | fa:16:3e:94:5d:70 | {"subnet_id": "b193dc42  |
    | be1b06fc6892             |      |                          |                   | -77ac-                   |
    |                          |      |                          |                   | 44cb-8622-f15331cf6319", |
    |                          |      |                          |                   | "ip_address":            |
    |                          |      |                          |                   | "172.24.4.12"}           |
    +--------------------------+------+--------------------------+-------------------+--------------------------+
    ```

* From the output above, we know that the router instance id associated with net1 default gateway is *bfc87e1e-6b93-446f-a305-6dc6153c1198* 

* Let's check interface list and routing table on the matching network namespace

    * interface list

        ```
        network-node# ip netns exec qrouter-bfc87e1e-6b93-446f-a305-6dc6153c1198 ip a                          
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1                                                      
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00                                                                       
            inet 127.0.0.1/8 scope host lo                                                                                              
               valid_lft forever preferred_lft forever                                                                                  
            inet6 ::1/128 scope host                                                                                                    
               valid_lft forever preferred_lft forever                                                                                  
        13: qr-2c628d36-76: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN qlen 1000                            
            link/ether fa:16:3e:ca:72:d8 brd ff:ff:ff:ff:ff:ff                                                                          
            inet 172.19.1.1/24 brd 172.19.1.255 scope global qr-2c628d36-76                                                             
               valid_lft forever preferred_lft forever                                                                                  
            inet6 fe80::f816:3eff:feca:72d8/64 scope link                                                                               
               valid_lft forever preferred_lft forever                                                                                  
        16: qg-a1b3323f-35: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN qlen 1000                            
            link/ether fa:16:3e:94:5d:70 brd ff:ff:ff:ff:ff:ff                                                                          
            inet 172.24.4.12/24 brd 172.24.4.255 scope global qg-a1b3323f-35                                                            
               valid_lft forever preferred_lft forever                                                                                  
            inet 172.24.4.3/32 brd 172.24.4.3 scope global qg-a1b3323f-35                                                               
               valid_lft forever preferred_lft forever                                                                                  
            inet6 fe80::f816:3eff:fe94:5d70/64 scope link                                                                               
               valid_lft forever preferred_lft forever
        ```

    * routing table

        ```
        # ip netns exec qrouter-bfc87e1e-6b93-446f-a305-6dc6153c1198 ip r       
        default via 172.24.4.1 dev qg-a1b3323f-35                                                                    
        172.19.1.0/24 dev qr-2c628d36-76  proto kernel  scope link  src 172.19.1.1                                   
        172.24.4.0/24 dev qg-a1b3323f-35  proto kernel  scope link  src 172.24.4.12                                      
        ```

    * NAT table

        ```
        network-node# ip netns exec qrouter-bfc87e1e-6b93-446f-a305-6dc6153c1198 iptables -L -t nat 
         
        Chain PREROUTING (policy ACCEPT)                                                                             
        target     prot opt source               destination                                                         
        neutron-l3-agent-PREROUTING  all  --  anywhere             anywhere                                          

        Chain INPUT (policy ACCEPT)                                                                                  
        target     prot opt source               destination                                                         

        Chain OUTPUT (policy ACCEPT)                                                                                 
        target     prot opt source               destination                                                         
        neutron-l3-agent-OUTPUT  all  --  anywhere             anywhere                                              

        Chain POSTROUTING (policy ACCEPT)                                                                            
        target     prot opt source               destination                                                         
        neutron-l3-agent-POSTROUTING  all  --  anywhere             anywhere                                         
        neutron-postrouting-bottom  all  --  anywhere             anywhere                                           

        Chain neutron-l3-agent-OUTPUT (1 references)                                                                 
        target     prot opt source               destination                                                         
        DNAT       all  --  anywhere             172.24.4.3         to:172.19.1.3                                  

        Chain neutron-l3-agent-POSTROUTING (1 references)                                                            
        target     prot opt source               destination                                                         
        ACCEPT     all  --  anywhere             anywhere             ! ctstate DNAT                                 

        Chain neutron-l3-agent-PREROUTING (1 references)                                                             
        target     prot opt source               destination                                                         
        DNAT       all  --  anywhere             172.24.4.3         to:172.19.1.3                                  
        REDIRECT   tcp  --  anywhere             169.254.169.254      tcp dpt:http redir ports 9697                  

        Chain neutron-l3-agent-float-snat (1 references)                                                             
        target     prot opt source               destination                                                         
        SNAT       all  --  172.19.1.3           anywhere             to:172.24.4.3   
             
        Chain neutron-l3-agent-snat (1 references)                                                                   
        target     prot opt source               destination                                                         
        neutron-l3-agent-float-snat  all  --  anywhere             anywhere                                          
        SNAT       all  --  anywhere             anywhere             to:172.24.4.12                                 
        SNAT       all  --  anywhere             anywhere             mark match ! 0x2/0xffff ctstate DNAT to:172.24.
        4.12                                                                                                         

        Chain neutron-postrouting-bottom (1 references)                                                              
        target     prot opt source               destination                                                         
        neutron-l3-agent-snat  all  --  anywhere             anywhere             /* Perform source NAT on outgoing traffic.*/                                                                                                   
        ```



* From the output above we know that 

    * after the packet enter the namespace via interface *qr-2c628d36-76*, it will be routed to the external default gateway via interface *qg-a1b3323f-35*
    
    * the neutron router in this case will also perform NAT
        
        * packet from VM with IP 172.19.1.3 will get static SNAT to 172.24.4.3
            > this is the floating IP associated to VM 'c-private'

        * Packets from other VM will have many to 1 NAT to 172.24.4.12

    > Note: it is possible to disable NAT completely on neutron router, in this case no NAT is required and we can assign public IP directly to the VM.


* Great! Source IP to our packet of interest is translated to "public IP" now
    * note: 'public' here means this IP is known/reachable from outside the openstack.


* One last step find out how the packet sent to the external network

* The packet was sent out from neutron namespace via interface *qg-a1b3323f-35*

* This interface is connected back to openvswitch bridge br-int but with different internal vlan tag. For clarity, i copy again the relevant section from ovs-vsctl output above

    * ovs-vsctl output

        ```
        Bridge br-ex                                                                                                                
            Controller "tcp:127.0.0.1:6633"                                                                                         
                is_connected: true                                                                                                  
            fail_mode: secure                                                                                                       
            Port br-ex                                                                                                              
                Interface br-ex                                                                                                     
                    type: internal                                                                                                  
            Port phy-br-ex                                                                                                          
                Interface phy-br-ex                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=int-br-ex}                                                                                       

        Bridge br-int                                                                                                               
            Port int-br-ex                                                                                                          
                Interface int-br-ex                                                                                                 
                    type: patch                                                                                                     
                    options: {peer=phy-br-ex}                                                                                       
            Port "qg-a1b3323f-35"                                                                                                   
                tag: 4                                                                                                              
                Interface "qg-a1b3323f-35"                                                                                          
                    type: internal                                                                                                  

        ```


    * br-int mac-address table

        ```
        network-node# ovs-appctl fdb/show br-int
         port  VLAN  MAC                Age
            3     3  fa:16:3e:3b:53:26    1
            8     3  fa:16:3e:ca:72:d8    1
            1     4  ae:05:ca:a5:1e:4d    1
           11     4  fa:16:3e:94:5d:70    1
        ```



* And here are some additional information related to br-ex bridge and external gateway

    * ovs-ofctl list-ports output
        
        ```
        network-node# ovs-ofctl show br-int
         ...
         1(int-br-ex): addr:de:88:5e:32:fc:16
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max
         11(qg-a1b3323f-35): addr:00:00:00:00:60:ca
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max



        network-node# ovs-ofctl show br-ex                                                  
         1(phy-br-ex): addr:c2:0f:89:c0:78:90                                                                        
             config:     0                                                                                           
             state:      0                                                                                           
             speed: 0 Mbps now, 0 Mbps max                                                                           
         LOCAL(br-ex): addr:ae:05:ca:a5:1e:4d                                                                        
             config:     0                                                                                           
             state:      0                                                                                           
             speed: 0 Mbps now, 0 Mbps max                                                                           

        ```

    * ovs flow dumps on br-int

        ```
        network-node# ovs-ofctl dump-flows br-int
         cookie=0x86954af3f33dce38, duration=223373.954s, table=0, n_packets=152182, n_bytes=14515209, idle_age=0, hard_age=65534, priority=3,in_port=1,vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:4,NORMAL
         cookie=0x86954af3f33dce38, duration=223464.136s, table=0, n_packets=29, n_bytes=1530, idle_age=65534, hard_age=65534, priority=2,in_port=1 actions=drop
         cookie=0x86954af3f33dce38, duration=223464.386s, table=0, n_packets=463806, n_bytes=43847428, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
        ```

    * ovs flow dumps on br-ex

        ```
        network-node# ovs-ofctl dump-flows br-ex                                            
        cookie=0x97910c7dc69d33fb, duration=265698.006s, table=0, n_packets=179268, n_bytes=17096370, idle_age=19, hard_age=65534, priority=0 actions=NORMAL
        ```

    * br-ex mac address table

        ```
        network-node# ovs-appctl fdb/show br-ex                                             
        port  VLAN  MAC                Age                                                                          
            1     0  fa:16:3e:94:5d:70   29                                                                          
        LOCAL     0  ae:05:ca:a5:1e:4d   29   
        ```


* And here are some additional setup specific on this lab

    * External default gateway : 172.24.4.1
    * MAC address of external default gateway: ae:05:ca:a5:1e:4d


* OK, what we can learn from output above

    * Packet received by ovs br-int port *qg-a1b3323f-35* (port 11) will hit the last rule in table 0 which has action NORMAL.
        * from the config we also know that this packet will get internal vlan tag = 4
    * OVS will do mac address lookup on br-int for the default gateway and find the outgoing port is ovs port 1 which is the patch interface to br-ex bridge *int-br-ex*
    * On br-ex table 0, ovs will simply use mac address table to decide where to forward the packet
    * In this case, the destination outgoing port is LOCAL
        * this is a special case for my setup, in most/normal cases the outgoing interface would be the real physical NIC on the network node server.
        * In my setup, i only have one NIC on network node, so i route the traffic from br-ex to the server eth0 interface. Basically i am using my network node as a router that connect Openstack and outside world.


* So, to summarize, our packet flow is complete

    ```
    @compute node
    c-private VM vNIC -- tapf7eae624-34 -- bridge qbrf7eae624-34 -- qvbf7eae624-34 -- qvof7eae624-34 -- ovs br-int -- patch-tun -- patch-int -- ovs br-tun -- vxlan-c0a8010c -- vxlan vni 90

    @network node
    vxlan vni 90 -- vxlan-c0a8010d -- ovs br-tun -- patch-int -- patch-tun -- ovs br-int -- qr-2c628d36-76 -- namespace qrouter-bfc87e1e-6b93-446f-a305-6dc6153c1198 -- qg-a1b3323f-35 -- ovs br-int -- patch-int-br-ex -- br-ex -- external NIC (in normal setup, not in this lab) 
    ```



So, that's it for the nort-south traffic from the VM to the external network. The same flow is applied for traffic from external network to the VM, but in the reverse direction.

For east-west traffic, the concept also similar, the only difference is, the packet is sent directly to the other network node if both source and destination are in the same virtual network, otherwise, the packet will be sent to the network node for layer 3 routing.

> Please note that this flow is specific to Neutron with Openvswitch plugin. Other plugin may have completely different flow. 



