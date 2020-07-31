---
layout: post
comments: true
title: "Calico and Kubernetes - Part 3 - External Connectivity"
categories: blog
descriptions: This is the 3rd part of Kubernetes with Project Calico as the networking plugin blog series.
tags: 
  - kubernetes
  - docker
  - networking
  - calico
  - container
date: 2017-10-21T23:39:55-04:00
---


## Overview

In general, container can communicate with external network in this k8s + Calico scenario:
* the first opton is by doing NAT on the k8s node
    * This is the "default" behavior. Any outgoing traffic from the container is SNAT by the node/host. 
        * "Default" here is not only limited for Calico, but also default behavior for many other netwworking plugin including flanel. 
    * It is the simplest method with the main limitation:
        * need to configure port forwarding for each service hosted inside k8s container
        * In my opinion, this is the main problem that Project Calico want to solve.
* the other option is by doing end-to-end IP routing
    * Project Calico is trying to simplify the container connectivity by simply providing end to end L3 routing
    * This mean, container connectivity is not different compared to baremetal connectivity. 
    

In the previous post, we found out the traffic flow between one container to the other container within the same k8s cluster. Now we are going to connect the k8s cluster to the external network. 

As we know, and to summarize the previous finding:

* by default, each k8s node has full mesh BGP peering relationship between each other. 
* Calico IPAM by default split the IP pool range to multiple /26 subnet. 
* Once a node need to run a container from a specific IP Pool, /26 will be assigned to that node. 
* Each node advertises its local container subnet to other nodes via BGP
    * if Calico cni is used with non Calico IPAM, each node may advertise each container IP as /32 route. 


So, our next step is to connect the k8s cluster to the external network via BGP. 

The definition of "external network" here is whatever the k8s node uplink connection connected to. 
* If k8s node is a baremetal node, most likely the IP gateway for the node is either layer 3 TOR switch or access router. 
* If k8s node is a VM, then the gateway would be the VM hypervisor
    * this is what i have in my test setup, each k8s node here is a VM inside Openstack with Contrail as networking plugin
    * In our case, each k8s node need to advertise its container prefix(es) to Contrail via BGP.
    * Fortunately, Contrail does have BGP as a service feature. 
        * Contrail BGPaaS and Calico has been briefly covered in [Contrail BGPaaS with Docker and Calico](Contrail-BGPaaS-with-Docker-and-Calico)


## Target topology

The following diagram is similar as the one in step 2.

```
                     Internet
                         +                         
                         |                        +-------------+ 
                   +---------------+              | vmx gateway | 
                   | Internet GW   |              |             | 
                   +---------------+              +-------------+ 
             192.168.1.1 |                                 | 192.168.1.22 
                         |                                 |
   +---+-----------------+--------------+------------------+-------------------------------------------------------------------+--------+
       |                                |                                                                                      |
192.168.1.19                       192.168.1.18                                                                         192.168.1.142
       |                                |                                                                                      |
       |                                |                                                                                      |
  +----+----+   +------------------------------------------------------------------------------------------------+        +----+-------+
  |Contrail |   |                       |                                                                        |        |  Test PC   |
  |Control  |   |                     vrouter                                                                    |        |            |
  +---------+   |                       |                                                                        |        +------------+
                |                       |               openstack net1 100.64.1.0/24                             | 
                |    +-------+----------+------------------------------------------------------------------+     |
                |            |                                              |                                    |
                |            |                                              |                                    |
                |       100.64.1.23  ubuntu-4 k8s node                 100.64.1.24  ubuntu-3 k8s node            |
                |      +-----+------------------------------+         +-----+------------------------------+     |
                |      |     |                              |         |     |                              |     |
                |      |     |     10.201.0.192/26          |         |     |     10.201.0.128/26          |     |
                |      |   +-+----------------+--------+    |         |   +-+----------------+--------+    |     |
                |      |     |                |             |         |     |                |             |     |
                |      |     |           .196 |             |         |     |           .130 |             |     |
                |      |     |     +-----------------+      |         |     |     +-----------------+      |     |
                |      |     |     |                 |      |         |     |     |                 |      |     |
                |      |     |     |  container 11   |      |         |     |     |  container 21   |      |     |
                |      |     |     +-----------------+      |         |     |     +-----------------+      |     |
                |      |     |                              |         |     |                              |     |
                |      |     |                              |         |     |                              |     |
                |      |     |     10.91.2.0/26             |         |     |       10.91.1.128/26         |     |
                |      |   +-+----------------+--------+    |         |   +-+----------------+--------+    |     |
                |      |     |                |             |         |     |                |             |     |
                |      |     |             .0 |             |         |     |           .128 |             |     |
                |      |     |     +-----------------+      |         |     |     +-----------------+      |     |
                |      |     |     |                 |      |         |     |     |                 |      |     |
                |      |     |     |  container 12   |      |         |     |     |  container 22   |      |     |
                |      |     |     +-----------------+      |         |     |     +-----------------+      |     |
                |      |                                    |         |                                    |     |
                |      |                                    |         |                                    |     |
                |      +------------------------------------+         +------------------------------------+     |
                |                                                                                                |
                | Compute node                                                                                   |
                +------------------------------------------------------------------------------------------------+

```

### Components

* k8s node 1: 
	* IP: 100.64.1.23
	* hostname: ubuntu-4
	* role: k8s master and worker node
* k8s node 2: 
	* IP: 100.64.1.24
	* hostname: ubuntu-3
	* role: worker node

* Notes:
	Although my test setup will have k8s nodes running as a VM on top of Openstack, it is not a mandatory requirement. You can have k8s on baremetal and directly connected to physical L2/L3 switches.



## Establish BGP Peering between each k8s node to Contrail control node

### Setup the connection

* on k8s side, use calictl to configure new BGP peering

    * go to inside calicoctl pod

        ```
        ubuntu@ubuntu-4:~$ kubectl exec -ti -n kube-system calicoctl -- /bin/busybox sh
        ```

    * create yaml file for BGP definistion

        ```
        ~ # cat >> bgp.yaml << EOF
        > apiVersion: v1
        > kind: bgpPeer
        > metadata:
        >   peerIP: 100.64.1.1
        >   scope: global
        > spec:
        >   asNumber: 64512
        > EOF
        ```
        
            * Notes: 
                * Contrail default GW for sbnet where the k8s node reside is 100.64.1.1
                * Contrail ASN is 64512

    * create the new BGP peers
    
        ```
        ~ # calicoctl create -f bgp.yaml
        Successfully created 1 'bgpPeer' resource(s)

        ```
    
    * Verify the new config
    
        ```
        ~ # calicoctl get -o yaml bgppeer
        - apiVersion: v1
          kind: bgpPeer
          metadata:
            peerIP: 100.64.1.1
            scope: global
          spec:
            asNumber: 64512
        ~ #
        ```


* On Contrail control node, use the Contrail GUI to configure BGP peering
    * Go to Contrail UI -> Configure -> Services -> BGP as a Service
    * Add new peer, and fill in all the necessart input.
    * Verify the new BGP peering on contrail 
        * go to Monitor -> Infrastructure -> Control nodes -> select the control node
    * More detailed steps to configure and verify BGP on Contrail side can be found in the previous post about [Contrail BGPaaS with Docker and Calico](Contrail-BGPaaS-with-Docker-and-Calico)
        

* Verify BGP Peering status from each node

    * For this, we need to get the calicoctl binary on each node
    
        ```
        wget https://github.com/projectcalico/calico/releases/download/v2.6.2/release-v2.6.2.tgz
        tar -xzvf release-v2.6.2.tgz
        cp release-v2.6.2/bin/calicoctl .
        rm -rf release-v2.6.2
        ```

    * run calicoctl on ubuntu-4 node

        ```
        ubuntu@ubuntu-4:~$ sudo ./calicoctl node status
        Calico process is running.

        IPv4 BGP status
        +--------------+-------------------+-------+------------+--------------------------------+
        | PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |              INFO              |
        +--------------+-------------------+-------+------------+--------------------------------+
        | 100.64.1.1   | global            | up    | 2017-10-21 | Established                    |
        | 100.64.1.24  | node-to-node mesh | up    | 2017-10-21 | Established                    |
        +--------------+-------------------+-------+------------+--------------------------------+

        IPv6 BGP status
        No IPv6 peers found.
        ```

    * ubuntu-3 node

        ```
        ubuntu@ubuntu-3:~$  sudo ./calicoctl node status
        Calico process is running.

        IPv4 BGP status
        +--------------+-------------------+-------+------------+--------------------------------+
        | PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |              INFO              |
        +--------------+-------------------+-------+------------+--------------------------------+
        | 100.64.1.1   | global            | up    | 2017-10-21 | Established                    |
        | 100.64.1.23  | node-to-node mesh | up    | 2017-10-21 | Established                    |
        +--------------+-------------------+-------+------------+--------------------------------+

        IPv6 BGP status
        No IPv6 peers found.
        ```


### Verify the packet flow

* Verify if container subnets are already received by MX gateway

    ```
    rw@gw-01> show route table contrail-public.inet.0 

    contrail-public.inet.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
    + = Active Route, - = Last Active, * = Both

    0.0.0.0/0          *[Static/5] 2d 07:03:17
                        > to 100.64.0.1 via lt-0/0/10.11
    ...deleted..
    10.91.1.128/26     *[BGP/170] 00:08:56, localpref 100, from 192.168.1.19
                          AS path: 64500 I, validation-state: unverified
                        > via gr-0/0/10.32769, Push 51
    10.91.2.0/26       *[BGP/170] 00:08:56, localpref 100, from 192.168.1.19
                          AS path: 64500 I, validation-state: unverified
                        > via gr-0/0/10.32769, Push 51
    10.201.0.128/26    *[BGP/170] 00:08:56, localpref 100, from 192.168.1.19
                          AS path: 64500 I, validation-state: unverified
                        > via gr-0/0/10.32769, Push 51
    10.201.0.192/26    *[BGP/170] 00:08:56, localpref 100, from 192.168.1.19
                          AS path: 64500 I, validation-state: unverified
                        > via gr-0/0/10.32769, Push 51
    ...deleted...
    100.64.1.23/32     *[BGP/170] 2d 06:57:58, MED 100, localpref 200, from 192.168.1.19
                          AS path: ?, validation-state: unverified
                        > via gr-0/0/10.32769, Push 51
    100.64.1.24/32     *[BGP/170] 2d 06:57:58, MED 100, localpref 200, from 192.168.1.19
                          AS path: ?, validation-state: unverified
                        > via gr-0/0/10.32769, Push 49
    ...deleted...
    ```

    * From the above output we can see that Cailco IP Pool for container IP, 10.201.x.x and 10.91.x.x, have been received by external gateway
    * For more detailed routing verification please refer to [Contrail BGPaaS with Docker and Calico](2017-06-18-Contrail-BGPaaS-with-Docker-and-Calico.md)


* Now it's time to verify the connection to each container. Let's check what is the IP of each container and which node are they hosted.

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods -o wide
    NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
    ssh-server                1/1       Running   0          2d        10.91.1.128    ubuntu-3
    ssh-server3               0/1       Pending   0          1d        <none>         <none>
    ssh-server4               1/1       Running   0          1d        10.91.2.0      ubuntu-4
    sshd-1-84c4bf4558-284dj   1/1       Running   0          2d        10.201.0.197   ubuntu-4
    sshd-2-78f7789cc8-95srr   1/1       Running   0          2d        10.201.0.130   ubuntu-3
    sshd-3-6bb86d6bf8-bq4q8   1/1       Running   0          2d        10.201.0.131   ubuntu-3
    ```


* Verify external connection container hosted in node 1: ubuntu-4

    ```
    rw@gw-01> ping count 3 10.91.2.0
    PING 10.91.2.0 (10.91.2.0): 56 data bytes
    64 bytes from 10.91.2.0: icmp_seq=0 ttl=61 time=1.819 ms
    64 bytes from 10.91.2.0: icmp_seq=1 ttl=61 time=1.873 ms
    64 bytes from 10.91.2.0: icmp_seq=2 ttl=61 time=1.694 ms

    --- 10.91.2.0 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max/stddev = 1.694/1.795/1.873/0.075 ms

    rw@gw-01> traceroute no-resolve 10.91.2.0 
    traceroute to 10.91.2.0 (10.91.2.0), 30 hops max, 40 byte packets
     1  100.64.0.2  1.550 ms  0.500 ms  0.750 ms
     2  * * *
     3  10.91.2.0  3.245 ms  1.641 ms  1.691 ms



    rw@gw-01> ping count 3 10.201.0.197        
    PING 10.201.0.197 (10.201.0.197): 56 data bytes
    64 bytes from 10.201.0.197: icmp_seq=0 ttl=61 time=2.708 ms
    64 bytes from 10.201.0.197: icmp_seq=1 ttl=61 time=2.130 ms
    64 bytes from 10.201.0.197: icmp_seq=2 ttl=61 time=75.739 ms

    --- 10.201.0.197 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max/stddev = 2.130/26.859/75.739/34.564 ms

    rw@gw-01> traceroute no-resolve 10.201.0.197 
    traceroute to 10.201.0.197 (10.201.0.197), 30 hops max, 40 byte packets
     1  100.64.0.2  1.311 ms  0.520 ms  0.976 ms
     2  * * *
     3  10.201.0.197  2.755 ms  1.937 ms  1.835 ms
    ```


* Verify external connection container hosted in node 1: ubuntu-3

```
rw@gw-01> ping 10.91.1.128 
PING 10.91.1.128 (10.91.1.128): 56 data bytes
^C
--- 10.91.1.128 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss

rw@gw-01> traceroute no-resolve 10.91.1.128 
traceroute to 10.91.1.128 (10.91.1.128), 30 hops max, 40 byte packets
 1  100.64.0.2  333.443 ms  0.695 ms  0.418 ms
 2  * * *
 3  * * *
 4  * *^C
```
    
    * Hmm, ping fail!


## Why container in ubuntu-3 can't communicate with external network ?

### Troubleshoot connectivity to container in node 2: ubuntu-3

* Check the route on mx gateway again. 

    ```
    rw@gw-01> show route 10.91.1.128 table contrail-public.inet.0 detail 

    contrail-public.inet.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
    10.91.1.128/26 (1 entry, 1 announced)
            *BGP    Preference: 170/-101
                    Route Distinguisher: 192.168.1.18:1
                    Next hop type: Indirect
                    Address: 0x9985ea8
                    Next-hop reference count: 15
                    Source: 192.168.1.19
                    Next hop type: Router, Next hop index: 629
                    Next hop: via gr-0/0/10.32769, selected
                    Label operation: Push 51
                    Label TTL action: prop-ttl
                    Load balance label: Label 51: None; 
                    Session Id: 0x4
                    Protocol next hop: 192.168.1.18
                    Label operation: Push 51
                    Label TTL action: prop-ttl
                    Load balance label: Label 51: None; 
                    Indirect next hop: 0x9814440 1048578 INH Session ID: 0x5
                    State: <Secondary Active Int Ext ProtectionCand>
                    Local AS: 64512 Peer AS: 64512
                    Age: 1d 10:05:40 	Metric2: 0 
                    Validation State: unverified 
                    Task: BGP_64512.192.168.1.19+14024
                    Announcement bits (1): 1-KRT 
                    AS path: 64500 I
                    Communities: target:64512:1001 target:64512:8000001 unknown iana 30c unknown iana 30c unknown type 8004 value fc00:7a1201 unknown type 8071 value fc00:4
                    Import Accepted
                    VPN Label: 51
                    Localpref: 100
                    Router ID: 192.168.1.19
                    Primary Routing Table bgp.l3vpn.0


    rw@gw-01> show dynamic-tunnels database 
    Table: inet.3

    ..deleted..

    Destination-network: 192.168.1.0/24
    Tunnel to: 192.168.1.18/32 State: Up
      Reference count: 10
      Next-hop type: gre
        Source address: 192.168.1.22
        Next hop: gr-0/0/10.32769
        State: Up

    ```
    
    * From the output above, routing from mx gateway to contrail compute node looks correct.
    * 10.91.1.128 point to next hop interface gr-0/0/10.32769 which is going to 192.168.1.18.
        * 192.168.1.18 is the IP of compute node where the k8s VM node is hosted.



### Is it a routing problem ?

* Verify routing table from Contrail control node 

    * Control node routing table to 10.91.1.128
    
        ![Contrail Control Node routing table ]({{site.baseurl}}/assets/images/contrail_control_node_ecmp.png)

    * Control node routing table to 10.91.1.128 via ubuntu-4

        ![Contrail Control Node routing table to Node 1 ]({{site.baseurl}}/assets/images/contrail_control_node_to_ubuntu_4.png)

    * Control node routing table to 10.91.1.128 via ubuntu-3

        ![Contrail Control Node routing table to Node 2]({{site.baseurl}}/assets/images/contrail_control_node_to_ubuntu_3.png)

    * From the output above, contrail control node received the same routes from both k8s node ubuntu-3 and ubuntu-4. 
        * Although the container is hosted by ubuntu-3, ubuntu-4 also advertised the same route because there is a full-mesh bgp peer between all the k8s node.
        * So, from control node point of view, everything looks OK.
        

* Find out which control-node route that actually used by data-plane    
    * In out setup, we kind of lucky, both ubuntu-3 and ubuntu-4 VM are hosted by Openstack compute-1 (192.168.1.18)
    * In this we can check the routing table on compute-1
    
        ![Contrail compute node routing table]({{site.baseurl}}/assets/images/contrail_compute_to_ubuntu_4.png)
        
    * OK, looks like packet going to 10.91.1.128 is sent thru ubuntu-4 node. 

* Based on the result above, in theory, ubuntu-4 should forward the packet to ubuntu-3 via IPIP tunnel between them. 

* Let's verify if ubuntu-3 node received the packet going to 10.91.1.128 container via IPIP tunnel

    ```
    ubuntu@ubuntu-3:~$ sudo tcpdump -n -i tunl0 icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
    15:41:38.483493 IP 100.64.0.1 > 10.91.1.128: ICMP echo request, id 62374, seq 13, length 64
    15:41:39.492471 IP 100.64.0.1 > 10.91.1.128: ICMP echo request, id 62374, seq 14, length 64
    15:41:40.502981 IP 100.64.0.1 > 10.91.1.128: ICMP echo request, id 62374, seq 15, length 64
    15:41:41.511815 IP 100.64.0.1 > 10.91.1.128: ICMP echo request, id 62374, seq 16, length 64
    15:41:42.522663 IP 100.64.0.1 > 10.91.1.128: ICMP echo request, id 62374, seq 17, length 64
    ^C
    ```

    * Yup, we can see the ping packet from gateway router to container 10.91.1.128 received on tunl0 interface.
    
* Now, let see if the packet is actually forwarded to veth interface between ubuntu-3 node and the container

    ```
    ubuntu@ubuntu-3:~$ ip r show 10.91.1.128
    10.91.1.128 dev cali90490b35e30  scope link 
    
    ubuntu@ubuntu-3:~$ sudo tcpdump -n -i cali90490b35e30 icmp
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on cali90490b35e30, link-type EN10MB (Ethernet), capture size 262144 bytes
    ^C
    0 packets captured
    0 packets received by filter
    0 packets dropped by kernel
    ```
    
    * Nope, nothing here. Something must blocking it.
    
* Verify if IP forwarding is enabled on ubuntu-3 node

    ```
    ubuntu@ubuntu-3:~$ sudo sysctl -a | grep forward | grep ipv4 | grep tunl0  
    net.ipv4.conf.tunl0.forwarding = 1
    net.ipv4.conf.tunl0.mc_forwarding = 0

    ubuntu@ubuntu-3:~$ sudo sysctl -a | grep forward | grep ipv4 | grep cali90490b35e30
    net.ipv4.conf.cali90490b35e30.forwarding = 1
    net.ipv4.conf.cali90490b35e30.mc_forwarding = 0
    ```

    * Looks good. Nothing wrong.
    
    
### Is it a firewall problem ?

* Maybe it is a firewall. Let's check IPTable

    ```
    ubuntu@ubuntu-3:~$ sudo iptables -L -n
    ```
    
    * Hmm, nothing obvious here.
    
* Let's check one more thing, reverse path forwarding rule.

    ```
    root@ubuntu-3:/home/ubuntu# sysctl -a | grep rp_filter
    net.ipv4.conf.all.arp_filter = 0
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.cali06827901978.arp_filter = 0
    net.ipv4.conf.cali06827901978.rp_filter = 1
    net.ipv4.conf.cali90490b35e30.arp_filter = 0
    net.ipv4.conf.cali90490b35e30.rp_filter = 1
    net.ipv4.conf.caliaf4e899510c.arp_filter = 0
    net.ipv4.conf.caliaf4e899510c.rp_filter = 1
    net.ipv4.conf.default.arp_filter = 0
    net.ipv4.conf.default.rp_filter = 1
    net.ipv4.conf.docker0.arp_filter = 0
    net.ipv4.conf.docker0.rp_filter = 1
    net.ipv4.conf.ens3.arp_filter = 0
    net.ipv4.conf.ens3.rp_filter = 1
    net.ipv4.conf.ens4.arp_filter = 0
    net.ipv4.conf.ens4.rp_filter = 1
    net.ipv4.conf.lo.arp_filter = 0
    net.ipv4.conf.lo.rp_filter = 0
    net.ipv4.conf.tunl0.arp_filter = 0
    net.ipv4.conf.tunl0.rp_filter = 1
    ```
    
    * Hmm, this could be why. The ping packet is asymmetric
        * incoming: mx gateway -> compute-1 -> ubuntu-4 node -(IPIP)-> ubuntu-3 node -> container 10.91.1.128
        * outgoing: container 10.91.1.128 -> ubuntu-3 node -> compute-1 -> mx gateway


### or it is something else ?

* Let's disable reverse path filtering

    ```
    root@ubuntu-3:/home/ubuntu# sysctl -w net.ipv4.conf.tunl0.rp_filter=0
    net.ipv4.conf.tunl0.rp_filter = 0
    root@ubuntu-3:/home/ubuntu# sysctl -w net.ipv4.conf.ens3.rp_filter=0
    net.ipv4.conf.ens3.rp_filter = 0
    root@ubuntu-3:/home/ubuntu# 
    ```
    
    * Yap, that's it !!!!
    
    * Now ping works
    
        ```
        rw@gw-01> ping count 3 10.91.1.128 
        PING 10.91.1.128 (10.91.1.128): 56 data bytes
        64 bytes from 10.91.1.128: icmp_seq=0 ttl=61 time=4.976 ms
        64 bytes from 10.91.1.128: icmp_seq=1 ttl=61 time=2.262 ms
        64 bytes from 10.91.1.128: icmp_seq=2 ttl=61 time=3.298 ms

        --- 10.91.1.128 ping statistics ---
        3 packets transmitted, 3 packets received, 0% packet loss
        round-trip min/avg/max/stddev = 2.262/3.512/4.976/1.118 ms

        rw@gw-01> traceroute no-resolve 10.91.1.128
        traceroute to 10.91.1.128 (10.91.1.128), 30 hops max, 40 byte packets
         1  100.64.0.2  0.831 ms  0.452 ms  1.320 ms
         2  * * *
         3  100.64.1.24  3.203 ms  2.408 ms  2.353 ms
         4  10.91.1.128  2.561 ms  2.138 ms  1.882 ms

        rw@gw-01>         
        ```
        

## Links
* [Kubernetes and Calico Part 1](Calico-and-Kubernetes-part-1)
* [Kubernetes and Calico Part 2](Calico-and-Kubernetes-part-2)
* [Kubernetes and Calico Part 4](Calico-and-Kubernetes-part-4)

