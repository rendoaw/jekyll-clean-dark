---
layout: post
comments: true
title: "Calico and Kubernetes - Part 4 - IPv6 for Container"
categories: blog
descriptions: This is the 4th part of Kubernetes with Project Calico as the networking plugin blog series.
tags: 
  - kubernetes
  - docker
  - networking
  - calico
  - container
  - contrail
  - openstack
  - vmx
date: 2017-10-29T10:39:55-04:00
---


This time we will try to setup ipv6 connectivity to the container using Calico. 
Initially i am planning to use the same k8s nodes as we did in the previous post but realized that those nodes are not ipv6 enabled. 


## Diagram

```
                     Internet
                         +                         
                         |                        +-------------+ 
                   +---------------+              | vmx gateway | 
                   | Internet GW   |              |             | 
                   +---------------+              +-------------+ 
             192.168.1.1 |                                 | 192.168.1.22 
                         |                                 |
   +---+-----------------+--------------+------------------+--------------------------------------------------------------------------------------------+--------+
       |                                |                                                                                                               |
192.168.1.19                       192.168.1.18                                                                                                  192.168.1.142
       |                                |                                                                                                               |
       |                                |                                                                                                               |
  +----+----+   +-------------------------------------------------------------------------------------------------------------------------+        +----+-------+
  |Contrail |   |                       |                                                                                                 |        |  Test PC   |
  |Control  |   |                     vrouter                                                                                             |        |            |
  +---------+   |                       |                                                                                                 |        +------------+
                |                       |               openstack net1 100.64.1.0/24 3000:1:1::/64                                        | 
                |    +-------+----------+-----------------------------------------------------------------------------------------------+ |
                |            |                                              |                                           |                 |
                |            |                                              |                                           |                 |
                |       100.64.1.20  kube-1 k8s node                     100.64.1.22  kube-2 k8s node              100.64.1.25            |
                |       3000:1:1::3                                      3000:1:1::4                               3000:1:1::5
                |      +-----+------------------------------+         +-----+------------------------------+            |                 |
                |      |                                    |         |     |                              |       +---------------+      |
                |      |                                    |         |     |       10.91.101.192/26       |       |               |      |
                |      |                                    |         |   +-+----------------+--------+    |       |   vmx101      |      |
                |      |                                    |         |     |                |             |       |               |      |
                |      |                                    |         |     |           .192 |             |       +---------------+      |
                |      |                                    |         |     |     +-----------------+      |                              |
                |      |                                    |         |     |     |                 |      |                              |
                |      |                                    |         |     |     |  pod1           |      |                              |
                |      |                                    |         |     |     +-----------------+      |                              |
                |      |                                    |         |                                    |                              |
                |      |                                    |         |                                    |                              |
                |      +------------------------------------+         +------------------------------------+                              |
                |                                                                                                                         |
                | Compute node                                                                                                            |
                +-------------------------------------------------------------------------------------------------------------------------+

```


## Setup and Preparation

So, I am creating two brand new k8s nodes using the following steps:

* Enable IPv6 on Contrail virtual network where the new k8s VM nodes will be connected
* Since by default Ubuntu 16.04 does not have dhcpv6 client enabled, we need to manually run the dhcpv6 client (or change /etc/network/interfaces.d/50-cloud-init.cfg and reboot the VM)
* Setup k8s master node

    ```
    root@kube-1:/home/ubuntu# kubeadm init --pod-network-cidr=10.202.0.0/24 --token-ttl 0
    root@kube-1:/home/ubuntu# exit
    exit
    ubuntu@kube-1:~$ rm -rf .kube/
    ubuntu@kube-1:~$ mkdir -p $HOME/.kube
    ubuntu@kube-1:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    ubuntu@kube-1:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

* Download calico.yaml and adjust some IPv6 related attributes

    ```
    ubuntu@kube-1:~$ diff calico.yaml calico.yaml.orig
    32,33d31
    <             "assign_ipv4": "true",
    <             "assign_ipv6": "true",
    160d157
    <           #image: quay.io/calico/node:master
    185c182
    <               value: "10.202.0.0/24"
    ---
    >               value: "192.168.0.0/16"
    190c187
    <               value: "true"
    ---
    >               value: "false"
    200,203d196
    <             - name: IP6
    <               value: "autodetect"
    <             - name: IP6_AUTODETECT_METHOD
    <               value: "first-found"
    ```

    * references:
        * http://blog.michali.net/2017/02/11/using-kubeadm-and-calico-plugin-for-ipv6-addresses/
        * http://blog.michali.net/2017/02/23/updates-ipv6-with-kubeadm-and-calico/
        * http://blog.michali.net/2017/02/17/kubernetescalico-plugin-with-ipv6-on-bare-metal/
        * http://blog.michali.net/2017/02/22/ipv6-multi-node-on-bare-metal/
        * http://blog.michali.net/wp-content/uploads/2017/02/calico.yaml_.txt
        * https://docs.projectcalico.org/v1.5/getting-started/docker/tutorials/ipv6
        * https://www.projectcalico.org/enable-ipv6-on-kubernetes-with-project-calico/


* Install calico

    ```
    ubuntu@kube-1:~$ kubectl apply -f calico.yaml 
    ubuntu@kube-1:~$ kubectl taint nodes --all node-role.kubernetes.io/master-
    ubuntu@kube-1:~$ kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/calicoctl.yaml
    ubuntu@kube-1:~$ sudo sysctl -w net.ipv6.conf.all.forwarding=1
    ```

* Create a new IPv6 and (optionally) IPv4 pool

    ```
    ~ # vi ippool.yaml
    ~ # cat ippool.yaml
    - apiVersion: v1
    kind: ipPool
    metadata:
        cidr: 10.91.101.0/24
    spec:
        ipip:
        enabled: true
        mode: always
    - apiVersion: v1
    kind: ipPool
    metadata:
        cidr: 3000:1:101::/64
    spec:
        ipip:
        mode: always

    ~ # /calicoctl create -f ippool.yaml
    Successfully created 2 'ipPool' resource(s)
    ~ # /calicoctl get ippool -o wide
    CIDR                       NAT     IPIP
    10.202.0.0/24              true    true
    10.91.101.0/24             false   true
    3000:1:101::/64            false   false
    fd80:24e2:f998:72d6::/64   false   false

    ~ # exit
    ```


* Configure Calico BGP 

    ```
    ubuntu@kube-1:~$ kubectl exec -ti -n kube-system calicoctl -- /bin/busybox sh
    ~ # /calicoctl config get asnumber
    64512
    ~ # /calicoctl config set asnumber 64501
    ~ # /calicoctl config get asnumber
    64501


    ~ # vi bgppeer.yaml
    ~ #
    ~ # cat bgppeer.yaml
    - apiVersion: v1
    kind: bgpPeer
    metadata:
        peerIP: 100.64.1.1
        scope: global
    spec:
        asNumber: 64512
    - apiVersion: v1
    kind: bgpPeer
    metadata:
        peerIP: 3000:1:1::1
        scope: global
    spec:
        asNumber: 64512

    ~ # /calicoctl create -f bgppeer.yaml
    Successfully created 2 'bgpPeer' resource(s)
    ~ # /calicoctl get bgppeer -o wide
    SCOPE    PEERIP        NODE   ASN
    global   100.64.1.1           64512
    global   3000:1:1::1          64512

    ~ # exit
    ```


* (Optionally) How to check BIRD config inside calico node container

    ```
    ubuntu@kube-1:~$ kubectl get pods -o wide --all-namespaces | grep node
    kube-system   calico-node-2bqpm                          2/2       Running   0          4m        100.64.1.22     kube-2
    kube-system   calico-node-bj8jv                          2/2       Running   0          32m       100.64.1.20     kube-1

    ubuntu@kube-1:~$ kubectl exec --namespace kube-system -it calico-node-bj8jv -- /bin/busybox sh
    Defaulting container name to calico-node.
    Use 'kubectl describe pod/calico-node-bj8jv' to see all of the containers in this pod.
    / # ps -ef | grep bird
    86 root       0:00 runsv bird
    87 root       0:00 runsv bird6
    91 root       0:00 bird -R -s /var/run/calico/bird.ctl -d -c /etc/calico/confd/config/bird.cfg
    93 root       0:00 bird6 -R -s /var/run/calico/bird6.ctl -d -c /etc/calico/confd/config/bird6.cfg
    847 root       0:00 grep bird

    / # cd /etc/calico/confd/config/

    /etc/calico/confd/config # more bird.cfg

    /etc/calico/confd/config # more bird6.cfg

    ```


* On each k8s node, we can also check IPv4 and IPv6 status

    ```
    ubuntu@kube-1:~$ sudo ./calicoctl node status
    Calico process is running.

    IPv4 BGP status
    +--------------+-------------------+-------+----------+-------------+
    | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
    +--------------+-------------------+-------+----------+-------------+
    | 100.64.1.1   | global            | up    | 21:40:34 | Established |
    | 100.64.1.22  | node-to-node mesh | up    | 21:43:05 | Established |
    +--------------+-------------------+-------+----------+-------------+

    IPv6 BGP status
    +--------------+-------------------+-------+----------+-------------+
    | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
    +--------------+-------------------+-------+----------+-------------+
    | 3000:1:1::1  | global            | start | 21:40:31 | Connect     |
    | 3000:1:1::4  | node-to-node mesh | up    | 21:43:07 | Established |
    +--------------+-------------------+-------+----------+-------------+
    ```



## Verification

* Launch a pod

    ```
    ubuntu@kube-1:~$ cat pod1.yaml
    ---
    apiVersion: v1
    kind: Pod
    metadata:
    annotations:
        "cni.projectcalico.org/ipv4pools": "[\"10.91.101.0/24\"]"
        "cni.projectcalico.org/ipv6pools": "[\"3000:1:101::0/64\"]"
    name: pod1
    labels:
        app: busybox
    spec:
    containers:
        - name: sshd
        image: "rendoaw/ubuntu_netkit:16.04"

    ubuntu@kube-1:~$ kubectl create -f pod1.yaml

    ubuntu@kube-1:~$ kubectl get pods -o wide
    NAME      READY     STATUS    RESTARTS   AGE       IP              NODE
    pod1      1/1       Running   0          4h        10.91.101.192   kube-2
    ```


* Verify routing table on both k8s nodes

    * on kube-1 (master + worker)

    ```
    ubuntu@kube-1:~$ ip r
    default via 100.64.1.1 dev ens3
    10.91.101.192/26 via 100.64.1.22 dev tunl0  proto bird onlink
    blackhole 10.202.0.64/26  proto bird
    10.202.0.65 dev calia499eb70bde  scope link
    10.202.0.192/26 via 100.64.1.22 dev tunl0  proto bird onlink
    100.64.1.0/24 dev ens3  proto kernel  scope link  src 100.64.1.20
    172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 linkdown

    ubuntu@kube-1:~$ ip -6 r
    3000:1:1::3 dev ens3  proto kernel  metric 256  mtu 1400 pref medium
    3000:1:1::/64 dev ens3  proto kernel  metric 256  mtu 1400 pref medium
    3000:1:101:0:7e48:cff1:8353:e140/122 via fe80::5e00:100 dev ens3  proto bird  metric 1024  pref medium
    fd80:24e2:f998:72d6:fb13:7db7:9fcc:7e40 dev calia499eb70bde  metric 1024  pref medium
    blackhole fd80:24e2:f998:72d6:fb13:7db7:9fcc:7e40/122 dev lo  proto bird  metric 1024  error -22 pref medium
    fe80::/64 dev ens3  proto kernel  metric 256  mtu 1400 pref medium
    fe80::/64 dev calia499eb70bde  proto kernel  metric 256  pref medium
    default via fe80::5e00:100 dev ens3  proto ra  metric 1024  expires 8987sec hoplimit 64 pref medium
    ```


    * on kube-2 (worker)

    ```
    root@kube-2:/home/ubuntu# ip r
    default via 100.64.1.1 dev ens3
    10.91.101.192 dev calice0906292e2  scope link
    blackhole 10.91.101.192/26  proto bird
    unreachable 10.202.0.64/26  proto bird
    blackhole 10.202.0.192/26  proto bird
    100.64.1.0/24 dev ens3  proto kernel  scope link  src 100.64.1.22
    172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 linkdown

    root@kube-2:/home/ubuntu# ip -6 r
    3000:1:1::4 dev ens3  proto kernel  metric 256  mtu 1400 pref medium
    3000:1:1::/64 dev ens3  proto kernel  metric 256  mtu 1400 pref medium
    3000:1:101:0:7e48:cff1:8353:e140 dev calice0906292e2  metric 1024  pref medium
    blackhole 3000:1:101:0:7e48:cff1:8353:e140/122 dev lo  proto bird  metric 1024  error -22 pref medium
    unreachable fd80:24e2:f998:72d6:fb13:7db7:9fcc:7e40/122 dev lo  proto bird  metric 1024  error -113 pref medium
    fe80::/64 dev ens3  proto kernel  metric 256  mtu 1400 pref medium
    fe80::/64 dev calice0906292e2  proto kernel  metric 256  pref medium
    ```


* Verify the networking inside container

    ```
    root@kube-2:/home/ubuntu# ssh root@10.91.101.192
    root@10.91.101.192's password:

    root@pod1:~# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
        link/ipip 0.0.0.0 brd 0.0.0.0
    4: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether be:7f:e9:4a:c3:67 brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 10.91.101.192/32 scope global eth0
        valid_lft forever preferred_lft forever
        inet6 3000:1:101:0:7e48:cff1:8353:e140/128 scope global
        valid_lft forever preferred_lft forever
        inet6 fe80::bc7f:e9ff:fe4a:c367/64 scope link
        valid_lft forever preferred_lft forever

    root@pod1:~# ip r
    default via 169.254.1.1 dev eth0
    169.254.1.1 dev eth0  scope link

    root@pod1:~# ip -6 r
    3000:1:101:0:7e48:cff1:8353:e140 dev eth0  proto kernel  metric 256  pref medium
    fe80::/64 dev eth0  proto kernel  metric 256  pref medium
    default via fe80::382d:fcff:fe52:46af dev eth0  metric 1024  pref medium
    root@pod1:~#
    ```



## Evrything works ? Not really ....

If we check the "calico node status" above, we can see that BGP peering between Calico and Contrail only works for IPv4 and not IPv6 peering.

    ```
    ubuntu@kube-1:~$ sudo ./calicoctl node status
    Calico process is running.

    IPv4 BGP status
    +--------------+-------------------+-------+----------+-------------+
    | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
    +--------------+-------------------+-------+----------+-------------+
    | 100.64.1.1   | global            | up    | 21:40:34 | Established |
    | 100.64.1.22  | node-to-node mesh | up    | 21:43:05 | Established |
    +--------------+-------------------+-------+----------+-------------+

    IPv6 BGP status
    +--------------+-------------------+-------+----------+-------------+
    | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
    +--------------+-------------------+-------+----------+-------------+
    | 3000:1:1::1  | global            | start | 21:40:31 | Connect     |
    | 3000:1:1::4  | node-to-node mesh | up    | 21:43:07 | Established |
    +--------------+-------------------+-------+----------+-------------+
    ```

After troubleshooting, I found out that Contrail does not support (or at least I don't know how) native IPv6 peering. Contrail is expecting IPv6 routes received over MP-BGP over IPv4 BGP peering session (similar as 6PE scenario). 



## Workaround for testing

Since now we can't test the external connectivity to/from the container outside Openstack cluster, as workaround, let test the connectivity between the container to other VM inside the same Openstack cluster. For this, I am run vMX (vmx101 in the diagram above) as Openstack tenant and setup BGP peering between vmx101 and both k8s nodes (kube-1 and kube-2). 



* vmx101 (100.64.1.25 3000:1:1::5)  config

    ```
    admin@vmx101# show protocols bgp group contrail
    type external;
    family inet {
        unicast;
    }
    family inet6 {
        unicast;
    }
    export nhs;
    peer-as 64512;
    neighbor 100.64.1.1;



    admin@vmx101# show protocols bgp group contrail-v6
    type external;
    family inet6 {
        unicast;
    }
    export nhs;
    peer-as 64512;
    neighbor 3000:1:1::1;
    ```

* vmx101 BGP status

    ```
    admin@vmx101# run show bgp summary
    Groups: 3 Peers: 6 Down peers: 1
    Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
    inet.0
                        23         15          0          0          0          0
    inet6.0
                        10          2          0          0          0          0
    Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
    100.64.1.20           64501       2229       2168       0       0    16:24:23 Establ
    inet.0: 0/3/3/0
    100.64.1.22           64501       2232       2169       0       0    16:24:24 Establ
    inet.0: 3/3/3/0
    3000:1:1::3           64501         11          8       0       0        3:15 Establ
    inet6.0: 0/2/2/0
    3000:1:1::4           64501       2227       2166       0       0    16:24:24 Establ
    inet6.0: 2/2/2/0



    admin@vmx101# run show route table inet6.0 protocol bgp

    inet6.0: 13 destinations, 16 routes (8 active, 0 holddown, 6 hidden)
    + = Active Route, - = Last Active, * = Both

    3000:1:101:0:7e48:cff1:8353:e140/122
                    *[BGP/170] 16:25:33, localpref 100
                        AS path: 64501 I, validation-state: unverified
                        > to 3000:1:1::4 via ge-0/0/2.0
                        [BGP/170] 00:04:24, localpref 100
                        AS path: 64501 I, validation-state: unverified
                        > to 3000:1:1::3 via ge-0/0/2.0
    ```

* k8s node status

    * kube-1

        ```
        ubuntu@kube-1:~$ sudo ./calicoctl node status
        Calico process is running.

        IPv4 BGP status
        +--------------+-------------------+-------+----------+-------------+
        | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
        +--------------+-------------------+-------+----------+-------------+
        | 100.64.1.1   | global            | up    | 21:40:34 | Established |
        | 100.64.1.22  | node-to-node mesh | up    | 21:43:05 | Established |
        | 100.64.1.25  | global            | up    | 22:08:54 | Established |
        +--------------+-------------------+-------+----------+-------------+

        IPv6 BGP status
        +--------------+-------------------+-------+----------+-------------+
        | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
        +--------------+-------------------+-------+----------+-------------+
        | 3000:1:1::1  | global            | start | 21:40:30 | Connect     |
        | 3000:1:1::4  | node-to-node mesh | up    | 21:43:06 | Established |
        | 3000:1:1::5  | global            | up    | 14:30:03 | Established |
        +--------------+-------------------+-------+----------+-------------+
        ```

    * kube-2

        ```
        ubuntu@kube-2:~$ sudo ./calicoctl node status
        Calico process is running.

        IPv4 BGP status
        +--------------+-------------------+-------+----------+-------------+
        | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
        +--------------+-------------------+-------+----------+-------------+
        | 100.64.1.20  | node-to-node mesh | up    | 21:43:06 | Established |
        | 100.64.1.1   | global            | up    | 21:43:09 | Established |
        | 100.64.1.25  | global            | up    | 22:08:54 | Established |
        +--------------+-------------------+-------+----------+-------------+

        IPv6 BGP status
        +--------------+-------------------+-------+----------+-------------+
        | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
        +--------------+-------------------+-------+----------+-------------+
        | 3000:1:1::3  | node-to-node mesh | up    | 21:43:07 | Established |
        | 3000:1:1::1  | global            | start | 21:43:05 | Connect     |
        | 3000:1:1::5  | global            | up    | 22:08:54 | Established |
        +--------------+-------------------+-------+----------+-------------+
        ```

* traceroute from vmx101 to container pod1

    ```
    admin@vmx101# run traceroute 3000:1:101:0:7e48:cff1:8353:e140 no-resolve
    traceroute6 to 3000:1:101:0:7e48:cff1:8353:e140 (3000:1:101:0:7e48:cff1:8353:e140) from 3000:1:1::5, 64 hops max, 12 byte packets
    1  3000:1:1::4  2.539 ms  1.640 ms  1.070 ms
    2  3000:1:101:0:7e48:cff1:8353:e140  2.154 ms  1.628 ms  1.496 ms



    admin@vmx101# run ping count 3 3000:1:101:0:7e48:cff1:8353:e140
    PING6(56=40+8+8 bytes) 3000:1:1::5 --> 3000:1:101:0:7e48:cff1:8353:e140
    16 bytes from 3000:1:101:0:7e48:cff1:8353:e140, icmp_seq=0 hlim=63 time=2.270 ms
    16 bytes from 3000:1:101:0:7e48:cff1:8353:e140, icmp_seq=1 hlim=63 time=1.683 ms
    16 bytes from 3000:1:101:0:7e48:cff1:8353:e140, icmp_seq=2 hlim=63 time=1.325 ms

    --- 3000:1:101:0:7e48:cff1:8353:e140 ping6 statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max/std-dev = 1.325/1.759/2.270/0.390 ms
    ```


Great! The result above prove that external IPv6 connectivity works for container. 



## Bonus testing

Since Calico can't advertise IPv6 routes to Contrail, I am using vMX to verify if Contrail BGP as a Service feature works for IPv6 too. 

* Configure vmx101 to advertise its directly connected IPv6 and other IPv6 prefixes received from k8s node to Contrail

    ```
    admin@vmx101# show interfaces lo0
    unit 0 {
        family inet6 {
            address 3000:1:1:aaaa::1/128;
        }
    }

    [edit]
    admin@vmx101# show protocols bgp group contrail
    type external;
    family inet {
        unicast;
    }
    family inet6 {
        unicast;
    }
    export [ nhs loopback ];
    peer-as 64512;
    neighbor 100.64.1.1;
    ```


* Verify BGP peering

    ```
    admin@vmx101# run show bgp summary
    Groups: 3 Peers: 6 Down peers: 1
    Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
    inet.0
                        23         15          0          0          0          0
    inet6.0
                        10          2          0          0          0          0
    Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
    100.64.1.1            64512       1997       2188       0       0    16:31:32 Establ
    inet.0: 12/17/17/0
    inet6.0: 0/6/0/0
    ```


* Verify if Contrail received the routes

    ![Contrail compute node routing table]({{site.baseurl}}/images/contrail_ipv6_routes.png)



* Verify if MX gateway receives IPv6 routes from vmx101 too 

    ```
    rw@gw-01# run show route table contrail-public.inet6.0

    contrail-public.inet6.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
    + = Active Route, - = Last Active, * = Both

    ...deleted...
    3000:1:1:aaaa::1/128
                    *[BGP/170] 00:02:56, localpref 100, from 192.168.1.19
                        AS path: 64510 I, validation-state: unverified
                        > via gr-0/0/10.32769, Push 73
    3000:1:101:0:7e48:cff1:8353:e140/122
                    *[BGP/170] 13:56:27, localpref 100, from 192.168.1.19
                        AS path: 64510 64501 I, validation-state: unverified
                        > via gr-0/0/10.32769, Push 73
    ...deleted...
    ```




## Summary

So in summary, 

* IPv6 for container with Calico as k8s CNI plugin works well. 
* Contrail BGP as a Service also works with IPv6 prefixes.
* The only thing that is not working (yet) on this setup is the interoperability between Calico and Contrail BGP as a Service feature. 
  * Calico is running separate BIRD daemon, 1 for IPv4 peering and 1 for IPv6 peering
  * Contrail expect a IPv6 unicast family over single MP-BGP peering over IPv4 BGP session


## Links

* [Kubernetes and Calico Part 1](Calico-and-Kubernetes-part-1)
* [Kubernetes and Calico Part 2](Calico-and-Kubernetes-part-2)
* [Kubernetes and Calico Part 3](Calico-and-Kubernetes-part-3)
