---
layout: post
comments: true
title: "Calico and Kubernetes - Part 2 - IP Pool and Intra Cluster Connectivity"
categories: blog
descriptions: This is the 2nd part of Kubernetes with Project Calico as the networking plugin blog series. This post will focus on IP address assignemnt and container to container connectivity.
tags: 
  - kubernetes
  - docker
  - networking
  - calico
  - container
date: 2017-10-20T23:39:55-04:00
---


## Overview

In this part 2, let's inspect our existing k8s setup in more detail especially from IP address alocation point of view. 


## Target topology

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
                |      |     |          subnet1a            |         |     |          subnet1b            |     |
                |      |   +-+----------------+--------+    |         |   +-+----------------+--------+    |     |
                |      |     |                |             |         |     |                |             |     |
                |      |     |                |             |         |     |                |             |     |
                |      |     |     +-----------------+      |         |     |     +-----------------+      |     |
                |      |     |     |                 |      |         |     |     |                 |      |     |
                |      |     |     |  container 11   |      |         |     |     |  container 21   |      |     |
                |      |     |     +-----------------+      |         |     |     +-----------------+      |     |
                |      |     |                              |         |     |                              |     |
                |      |     |                              |         |     |                              |     |
                |      |     |          subnet2a            |         |     |          subnet2b            |     |
                |      |   +-+----------------+--------+    |         |   +-+----------------+--------+    |     |
                |      |     |                |             |         |     |                |             |     |
                |      |     |                |             |         |     |                |             |     |
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



## Add more stuff on existing setup

### Let's bring up 2 more containers

* container number 2

    ```
    ubuntu@ubuntu-4:~$ kubectl run sshd-2 --image=rastasheep/ubuntu-sshd:16.04
    deployment "sshd-2" created
    ```

* container number 3

    ```
    ubuntu@ubuntu-4:~$ kubectl run sshd-3 --image=rastasheep/ubuntu-sshd:16.04
    deployment "sshd-3" created
    ```

* Verify the containers status

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods -o wide
    NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
    sshd-1-84c4bf4558-284dj   1/1       Running   0          1d        10.201.0.197   ubuntu-4
    sshd-2-78f7789cc8-95srr   1/1       Running   0          1d        10.201.0.130   ubuntu-3
    sshd-3-6bb86d6bf8-bq4q8   1/1       Running   0          1d        10.201.0.131   ubuntu-3
    ```

* Looks like we have 1 pod on ubuntu-4 host and 2 pods in ubuntu-3 host.

* All of the pods above were getting IP from the IP Pool 10.201.0.0/24 that specified during kubeadm init.



## Verify k8s node networking setup

### Find out k8s node networking setup

* Based on theory, we should have the following:

    * Calico by default will create full-mesh IPIP tunnel between each node.
    * Calico will create full-mesh BGP Peering between each node
    * Unlike in Openstack where each VM is advertised as single /32 route, in k8s or docker, Calico will do route aggregation on each node
        * Based on some examples done by other people, and also myself in https://rendoaw.github.io/2017/06/Contrail-BGPaaS-with-Docker-and-Calico, looks like Calico will aggregate the routes per /26 prefix.
        * Other reference: https://docs.projectcalico.org/v2.6/reference/private-cloud/l3-interconnect-fabric

* Let's go back to our setup

* Verify interface on the host. 

    * node 1: ubuntu-4

        ```
        ubuntu@ubuntu-4:~$ ip a
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            inet 127.0.0.1/8 scope host lo
               valid_lft forever preferred_lft forever
            inet6 ::1/128 scope host 
               valid_lft forever preferred_lft forever
        2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UP group default qlen 1000
            link/ether 02:f5:83:2b:70:b6 brd ff:ff:ff:ff:ff:ff
            inet 100.64.1.23/24 brd 100.64.1.255 scope global ens3
               valid_lft forever preferred_lft forever
            inet6 fe80::f5:83ff:fe2b:70b6/64 scope link 
               valid_lft forever preferred_lft forever
        3: ens4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
            link/ether 02:92:95:10:a4:d7 brd ff:ff:ff:ff:ff:ff
        4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
            link/ether 02:42:f0:fe:30:76 brd ff:ff:ff:ff:ff:ff
            inet 172.17.0.1/16 scope global docker0
               valid_lft forever preferred_lft forever
        5: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1
            link/ipip 0.0.0.0 brd 0.0.0.0
            inet 10.201.0.192/32 scope global tunl0
               valid_lft forever preferred_lft forever
        6: cali5bed1c02fcb@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
            link/ether 0e:75:a4:ac:a0:9b brd ff:ff:ff:ff:ff:ff link-netnsid 0
            inet6 fe80::c75:a4ff:feac:a09b/64 scope link 
               valid_lft forever preferred_lft forever
        7: calidd49badf8e9@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
            link/ether f6:02:04:52:7a:e4 brd ff:ff:ff:ff:ff:ff link-netnsid 1
            inet6 fe80::f402:4ff:fe52:7ae4/64 scope link 
               valid_lft forever preferred_lft forever
        ```

    * notes:
        * ens3 is host 1st NIC for uplink interface
        * ens4 is host 2nd NIC, not used
        * docker0 is the default bidge for docker. This is not used in Kubernetes+Calico.
        * tunl0 is the IPIP tunnel between nodes. 
        * cali....@..  is the interface between the host and the container.


    * node 2: ubuntu-3 interface list
    
        ```
        root@ubuntu-3:/home/ubuntu# ip a
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            inet 127.0.0.1/8 scope host lo
               valid_lft forever preferred_lft forever
            inet6 ::1/128 scope host 
               valid_lft forever preferred_lft forever
        2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UP group default qlen 1000
            link/ether 02:a2:f0:89:15:28 brd ff:ff:ff:ff:ff:ff
            inet 100.64.1.24/24 brd 100.64.1.255 scope global ens3
               valid_lft forever preferred_lft forever
            inet6 fe80::a2:f0ff:fe89:1528/64 scope link 
               valid_lft forever preferred_lft forever
        3: ens4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
            link/ether 02:e8:df:d6:11:b7 brd ff:ff:ff:ff:ff:ff
        4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
            link/ether 02:42:8f:0e:1c:cf brd ff:ff:ff:ff:ff:ff
            inet 172.17.0.1/16 scope global docker0
               valid_lft forever preferred_lft forever
        5: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1
            link/ipip 0.0.0.0 brd 0.0.0.0
            inet 10.201.0.128/32 scope global tunl0
               valid_lft forever preferred_lft forever
        6: cali06827901978@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
            link/ether f2:12:2c:d6:b4:99 brd ff:ff:ff:ff:ff:ff link-netnsid 0
            inet6 fe80::f012:2cff:fed6:b499/64 scope link 
               valid_lft forever preferred_lft forever
        7: caliaf4e899510c@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
            link/ether 3a:4b:96:92:d8:63 brd ff:ff:ff:ff:ff:ff link-netnsid 1
            inet6 fe80::384b:96ff:fe92:d863/64 scope link 
               valid_lft forever preferred_lft forever
        ```


* Verify routing table on the node

    * node 1: ubuntu-4

        ```
        ubuntu@ubuntu-4:~$ ip r
        default via 100.64.1.1 dev ens3 
        10.201.0.128/26 via 100.64.1.24 dev tunl0  proto bird onlink 
        blackhole 10.201.0.192/26  proto bird 
        10.201.0.196 dev cali5bed1c02fcb  scope link 
        10.201.0.197 dev calidd49badf8e9  scope link 
        100.64.1.0/24 dev ens3  proto kernel  scope link  src 100.64.1.23 
        172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 linkdown 
        ```

    * notes
        * 10.201.0.192/26 is allocated for this node (ubuntu-4). 
            * This is the route aggregation that mentioned earlier.
        * 10.201.0.128/26 is route aggregation that allocated for 2nd worker. 
            * see the next-hop is tunl0 interface with next-hop is ubuntu-3 IP address
        * 10.201.0.197 is the container IP address. This IP is reachable via calidd49badf8e9 interface.
        * 172.17.0.0/16 is default docker0 bridge interface. I believe it is not used at all in Kubernetes with Calico.


    * node 2: ubuntu-3

        ```
        root@ubuntu-3:/home/ubuntu# ip r
        default via 100.64.1.1 dev ens3 
        blackhole 10.201.0.128/26  proto bird 
        10.201.0.130 dev cali06827901978  scope link 
        10.201.0.131 dev caliaf4e899510c  scope link 
        10.201.0.192/26 via 100.64.1.23 dev tunl0  proto bird onlink 
        100.64.1.0/24 dev ens3  proto kernel  scope link  src 100.64.1.24 
        172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 linkdown 
        ```

    * notes
        * 10.201.0.128/26 is allocated for this node (ubuntu-3). 
            * This is the route aggregation that mentioned earlier.
        * 10.201.0.192/26 is route aggregation that allocated for other node (ubuntu-4)
            * see the next-hop is tunl0 interface with next-hop is ubuntu-4 IP address
        * 10.201.0.130 is the container IP address. This IP is reachable via cali06827901978 interface.
        * 172.17.0.0/16 is default docker0 bridge interface. I believe it is not used at all in Kubernetes with Calico.

    

### Find out how the network is setup is inside container

* Let's go inside the container. 

* It was on purpose to choose sshd image as a test container because it will make us easier to go to inside the container and do ping, traceroute or other related networking test

* Default credential for the ssh container image is root/root. IP address of each container can be found via te following command

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods -o wide
    NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
    sshd-1-84c4bf4558-284dj   1/1       Running   0          1d        10.201.0.197   ubuntu-4
    sshd-2-78f7789cc8-95srr   1/1       Running   0          1d        10.201.0.130   ubuntu-3
    sshd-3-6bb86d6bf8-bq4q8   1/1       Running   0          1d        10.201.0.131   ubuntu-3
    ```

* Let's ssh to sshd-1 container

    ```
    ubuntu@ubuntu-4:~$ ssh root@10.201.0.197
    The authenticity of host '10.201.0.197 (10.201.0.197)' can't be established.
    ECDSA key fingerprint is SHA256:oeXGFyX/mdzuCxeulD1bOzDe1lCbIktPxb7vKNPiDOs.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '10.201.0.197' (ECDSA) to the list of known hosts.
    ```

    * Since this container is very minimum, let's install some basic packages.

    ```
    root@sshd-1-84c4bf4558-284dj:/# apt-get update 
    root@sshd-1-84c4bf4558-284dj:/# apt-get install iproute 2
    root@sshd-1-84c4bf4558-284dj:~# apt-get install inetutils-ping traceroute
    ```

* Now, we verify the interface and routing table

    ```
    root@sshd-1-84c4bf4558-284dj:/sbin# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
        link/ipip 0.0.0.0 brd 0.0.0.0
    4: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether ea:ff:92:a4:32:63 brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 10.201.0.197/32 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::e8ff:92ff:fea4:3263/64 scope link 
           valid_lft forever preferred_lft forever

    root@sshd-1-84c4bf4558-284dj:~# ip r
    default via 169.254.1.1 dev eth0
    169.254.1.1 dev eth0  scope link
    ```

    * Notes:
        * container eth0 basically is the other side of the veth pair that terminated on the host.
        * container eth0 IP is coming from the Calico IP Pool
        * container IP is /32 
        * default gateway is eth0 interface. 


* Let's do traceroute to other container that reside in other node

    ```
    root@sshd-1-84c4bf4558-284dj:~# traceroute -n 10.201.0.130
    traceroute to 10.201.0.130 (10.201.0.130), 30 hops max, 60 byte packets
     1  100.64.1.23  0.158 ms  0.060 ms  0.055 ms
     2  10.201.0.128  0.960 ms  0.769 ms  0.662 ms
     3  10.201.0.130  0.648 ms  0.625 ms  0.507 ms
    ```

    * from the above, the 1st hop is ubuntu-4 which is the host of sshd1 container
    * then, the 2nd hop is tunl0 IP on ubuntu-3 which is the host of sshd2 container
    

* ok, just for curiosity, let's traceroute to internet

    ```
    root@sshd-1-84c4bf4558-284dj:~# traceroute -n 8.8.8.8     
    traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
     1  100.64.1.23  0.112 ms  0.039 ms  0.075 ms
     2  100.64.1.1  0.765 ms  0.645 ms  0.485 ms
     3  100.64.0.2  2.057 ms  2.880 ms  3.342 ms
     4  100.64.0.1  8.384 ms  8.313 ms  8.239 ms
     5  192.168.1.1  9.304 ms  9.188 ms  9.199 ms
     6  * * *
     7  67.59.226.53  16.035 ms  17.296 ms  17.308 ms
     8  67.83.251.141  22.326 ms 67.83.251.133  21.483 ms  24.335 ms
     9  65.19.114.67  25.486 ms 67.59.239.119  20.946 ms 65.19.114.67  21.563 ms
    10  64.15.0.8  21.668 ms 64.15.1.64  20.923 ms 64.15.1.90  22.351 ms
    11  72.14.215.203  22.273 ms  16.582 ms *
    12  * * *
    13  209.85.243.19  12.440 ms 108.170.238.203  16.062 ms 108.170.228.135  19.197 ms
    14  8.8.8.8  19.882 ms  20.018 ms  20.675 ms
    ```

    * Woa, the container can access the internet!


### WAIT, something looks suspicious

* We have not configure any route from outside network to the container, how come the container can access internet and do apt-get?

    * The answer is because the default IP Pool has NAT enabled. We can verify that as below:
    
        ```
        ubuntu@ubuntu-4:~$ kubectl exec -ti -n kube-system calicoctl -- /calicoctl get -o yaml ippool
        - apiVersion: v1
          kind: ipPool
          metadata:
            cidr: 10.201.0.0/24
          spec:
            ipip:
              enabled: true
              mode: always
            nat-outgoing: true
        ```
    
    * That's mean the host will NAT all outgoing traffic from the container. 
    
    * We can also see the NAT from the IPTables below
    
        ```
        ubuntu@ubuntu-4:~$ sudo iptables -L -t nat -n
        ...
        Chain cali-nat-outgoing (1 references)
        target     prot opt source               destination         
        MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:Wd76s91357Uv7N3v */ match-set cali4-masq-ipam-pools src ! match-set cali4-all-ipam-pools dst
        ...


        root@ubuntu-4:/home/ubuntu# ipset list cali4-masq-ipam-pools 
        Name: cali4-masq-ipam-pools
        Type: hash:net
        Revision: 6
        Header: family inet hashsize 1024 maxelem 1048576
        Size in memory: 448
        References: 1
        Members:
        10.201.0.0/24
        ```

    * OK, one mystery is solved. 



## BUT .... i though the intention of using Calico instead of flannel is to have full end-to-end routing to each container without any NAT?

* Yes, but on that purpose either we need to disable NAT on default IP Pool, or we can create a new Pool with NAT disabled

* Let's create a new pool with IP Pool disable

* To make it esier to run the calicoctl utility, let's go inside calicoctl container and run the command from there

    ```
    ubuntu@ubuntu-4:~$ kubectl exec -ti -n kube-system calicoctl -- /bin/busybox sh
    ~ # pwd
    /root
    ```

* Prepare a yaml file to define a new pool. In this case, my new pool is 10.91.1.0/24

    ```
    ~ # cat ippool.yaml 
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.91.1.0/24
      spec:
        ipip:
          enabled: true
          mode: always
        nat-outgoing: false
    ```

* Enable new IP Pool

    ```
    / # /calicoctl create -f ippool.yaml
    Successfully created 1 'ipPool' resource(s)
    ```

* Verify all IP Pools

    ```
    / # /calicoctl get -o yaml ippool
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.201.0.0/24
      spec:
        ipip:
          enabled: true
          mode: always
        nat-outgoing: true
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.91.1.0/24
      spec:
        ipip:
          enabled: true
          mode: always
    / #
    ```

* Before we do any traffic test, since we know that Calico will do /26 aggregation, i am curious what will happen if i create IP Pool smaller that /26

    ```
    ~ # calicoctl create -f ippool-small.yaml 
    Failed to execute command: error with field CIDR = '10.91.2.0/27' (IP pool size is too small (min /26) for use with Calico IPAM)
    ```

* Nice.. smaller that /26 is not allowed. How about /26 itself

    ```
    ~ # cat ippool2.yaml 
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.91.2.0/26
      spec:
        ipip:
          enabled: true
          mode: always
        nat-outgoing: false


    / # /calicoctl create -f ippool2.yaml
    Successfully created 1 'ipPool' resource(s)

    
    ~ # cat ippool2.yaml 
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.91.2.0/26
      spec:
        ipip:
          enabled: true
          mode: always
        nat-outgoing: false
        
        
    ~ # calicoctl get ippool -o yaml
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.201.0.0/24
      spec:
        ipip:
          enabled: true
          mode: always
        nat-outgoing: true
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.91.1.0/24
      spec:
        ipip:
          enabled: true
          mode: always
    - apiVersion: v1
      kind: ipPool
      metadata:
        cidr: 10.91.2.0/26
      spec:
        ipip:
          enabled: true
          mode: always
    ```


* OK, now time to create a new container or pod, and this time we need to make sure that pod will get IP from the new pool. For this, we need to define the pod in yaml format before creating it.

    ```
    ubuntu@ubuntu-4:~$ more pod1.yaml 
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"10.91.1.0/24\"]" 
      name: ssh-server
      labels:
        app: sshd
    spec:
      containers:
        - name: ssh-server-container
          image: "rastasheep/ubuntu-sshd:16.04"
    ```

* Now, we create the pod

    ```
    ubuntu@ubuntu-4:~$ kubectl create -f pod1.yaml
    pod "ssh-server" created
    ```

* Verify the new Pod IP

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods -o wide
    NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
    ssh-server                1/1       Running   0          1d        10.91.1.128    ubuntu-3
    ...

    ```

* Verify the host - ubuntu-3 routing table

    ```
    root@ubuntu-3:/home/ubuntu# ip r
    default via 100.64.1.1 dev ens3 
    10.91.1.128 dev cali90490b35e30  scope link 
    blackhole 10.91.1.128/26  proto bird 
    ...
    ```
    
    * now we have a new aggregation route on ubuntu-3 which is 10.91.1.128/26
    * This also a proof that Calico will always split the IP Pool into multiple /26.


* Great, the POD now is using new Pool. We can go to inside he new pod, and do some ping to outside network and they will fail because this pool is not NATed.


## Just curious, what will happen with the other new IP Pool that only have /26 subnet size.

* Remember that we also create a second new IP Pool with /26 subnet size. In theory, if Calico split the IP Pool with /26 each, then in this case, only one node can have container with second new IP pool range (10.91.2.0/26). 

* For this test case, we need to find a way to pin a container to specific node. Why?
    * Because, if we can create 2 new pods, let say pod-A and pod-B, and pod-A is forced to be in node-1 and pod-B in node-2, then we can verify how Calico split the /26 IP Pool into multiple nodes.

* To pin a pod into a specific node, we need to use "node selector" feature. For that, we need to assign unique label for each node

    ```
    ubuntu@ubuntu-4:~$ kubectl label nodes ubuntu-4 node_id=n4
    node "ubuntu-4" labeled
    ubuntu@ubuntu-4:~$ kubectl label nodes ubuntu-3 node_id=n3
    node "ubuntu-3" labeled
    ```

    * we assign "n4" label to ubuntu-4 node and "n3" to ubuntu-3 node

* Create a YAML to define the container and its node pinning

    ```
    ubuntu@ubuntu-4:~$ cat pod_on_node3.yaml 
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"10.91.2.0/26\"]" 
      name: ssh-server3
      labels:
        app: sshd
    spec:
      containers:
        - name: ssh-server-container
          image: "rastasheep/ubuntu-sshd:16.04"
      nodeSelector:
        node_id: n3
    ubuntu@ubuntu-4:~$ 
    ubuntu@ubuntu-4:~$ 
    ubuntu@ubuntu-4:~$ 
    ubuntu@ubuntu-4:~$ cat pod_on_node4.yaml 
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"10.91.2.0/26\"]" 
      name: ssh-server4
      labels:
        app: sshd
    spec:
      containers:
        - name: ssh-server-container
          image: "rastasheep/ubuntu-sshd:16.04"
      nodeSelector:
        node_id: n4
    ```

* Create the Pods

    ```
    ubuntu@ubuntu-4:~$ kubectl create -f pod_on_node4.yaml
    ubuntu@ubuntu-4:~$ kubectl create -f pod_on_node3.yaml
    ```
    
    Notes: the new pods will have name ssh-server3 and ssh-server4

* Verify the status

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods -o wide
    NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
    ssh-server                1/1       Running   0          20h       10.91.1.128    ubuntu-3
    ssh-server3               0/1       Pending   0          25s       <none>         <none>
    ssh-server4               1/1       Running   0          1m        10.91.2.0      ubuntu-4
    sshd-1-84c4bf4558-284dj   1/1       Running   0          22h       10.201.0.197   ubuntu-4
    sshd-2-78f7789cc8-95srr   1/1       Running   0          21h       10.201.0.130   ubuntu-3
    sshd-3-6bb86d6bf8-bq4q8   1/1       Running   0          20h       10.201.0.131   ubuntu-3
    ```

* Wait for a while

* Interesting, ssh-server3 stay in pending status. Let's check more detail

    ```
    ubuntu@ubuntu-4:~$ kubectl describe pod ssh-server3
    Name:         ssh-server3
    Namespace:    default
    Node:         <none>
    Labels:       app=sshd
    Annotations:  cni.projectcalico.org/ipv4pools=["10.91.2.0/26"]
    Status:       Pending
    IP:           
    Containers:
      ssh-server-container:
        Image:        rastasheep/ubuntu-sshd:16.04
        Port:         <none>
        Environment:  <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-h52zb (ro)
    Conditions:
      Type           Status
      PodScheduled   False
    Volumes:
      default-token-h52zb:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-h52zb
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  node_id=n3
    Tolerations:     node.alpha.kubernetes.io/notReady:NoExecute for 300s
                     node.alpha.kubernetes.io/unreachable:NoExecute for 300s
    Events:
      Type     Reason            Age                From               Message
      ----     ------            ----               ----               -------
      Warning  FailedScheduling  26s (x7 over 57s)  default-scheduler  No nodes are available that match all of the predicates: MatchNodeSelector (2).
    ```

* So, ssh-server3 is pending because k8s scheduler can't find any host that satisfy the requirement which has "n3" label. Why?

    * it because 10.91.2.0/26 IP Pool can only be split into a single aggregated route. 
    * We created ssh-server4 first which is pinned to ubuntu-4 node. 
        * this way, 10.91.2.0/26 is allocated to ubuntu-4
    * When we create ssh-server3 which is pinned to ubuntu-3 node, no more sub-prefix is available from 10.91.2.0/26. 
        * Calico can't assign any aggregated route to ubuntu-3 node
        * This make ubuntu-3 is ineligible to host any container with 10.91.2.0/26 IP range.

    * Great! This is according my expectation. If number of sub-prefix chunk is smaller than number of the node, then the container for that IP Pool will be only placed into some of the available nodes.



* OK, we know for sure that Calico split the Pool into multiple /26 prefix. But which module/script is doing this?
    * After searching around, i found that the following module is responsible for the splitting, and /26 size is hardcoded there.
    * https://github.com/projectcalico/libcalico/blob/master/calico_containers/pycalico/block.py



OK, now we have more insight how the IP allocation works and know there are different behavior can be configured for different IP Pool. 


I'll continue with connectivity between the container and the external network in the next post here: [Kubernetes and Calico Part 3](2017-10-21-Calico-and-Kubernetes-part-3.md)



## Links
* [Kubernetes and Calico Part 1](2017-10-20-Calico-and-Kubernetes-part-1.md)
* [Kubernetes and Calico Part 3](2017-10-21-Calico-and-Kubernetes-part-3.md)
