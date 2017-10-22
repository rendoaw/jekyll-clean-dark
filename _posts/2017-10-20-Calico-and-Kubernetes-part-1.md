---
layout: post
comments: true
title: "Calico and Kubernetes - Part 1 - Setup"
categories: blog
descriptions: This is the first part of Kubernetes with Project Calico as the networking plugin blog series.
tags: 
  - kubernetes
  - docker
  - networking
  - calico
  - container
date: 2017-10-20T22:39:55-04:00
---

## Overview

This is the first part of Kubernetes with Project Calico as the networking plugin blog series.
The main driver for this experiment is to know in detail how is the packet flow works inside K8S with Calico as networking plugin.

I'll split the post into the following

* Part 1
	* This part will cover the steps to quickly setup the lab environment and get the k8s+Calico works. 
	* This is useful when we need to create and destroy the cluster several time to test different scenario
	* If you have existing k8s cluster with Calico already, then nothing special here

* Part 2
	* This part will focus on Calico IP address allocation and traffic flow between one container to the other container.
	* To jump to part to, you can go to [Kubernetes and Calico Part 2](2017-10-20-Calico-and-Kubernetes-part-2.md)

* Part 3
	* This part will focus on how to establish connectivity between container and external world.
	* To jump to part to, you can go to [Kubernetes and Calico Part 3](2017-10-20-Calico-and-Kubernetes-part-3.md)



## Installation and Configuration

There are many ways to setup K8S cluster. For this activity i am choosing Kubeadm because it seems the simplest way of creating multi-nodes k8s cluster, regardless we will run k8s on baremetal or VM.


### Install docker, kubeadm and other kubernetes tools. 

* Reference
Follow https://kubernetes.io/docs/setup/independent/install-kubeadm/

* Summary of the step from the URL above

    ```
    apt-get update && apt-get install -y curl apt-transport-https
    apt-get install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker

    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    ```

### Initialize Kubernetes

* Before we initialize k8s cluster, we need to decide and reserve the subnet range for container IP address pool.

* In the following example, i am using 10.201.0.0/24 as IP Pool for the container IP.

* Command example

    ```
    ubuntu@ubuntu-4:~$ sudo su  
    ubuntu@ubuntu-4:~# kubeadm init --pod-network-cidr=10.201.0.0/24 --token-ttl 0

    [kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
    [init] Using Kubernetes version: v1.8.1
    [init] Using Authorization modes: [Node RBAC]
    [preflight] Running pre-flight checks
    [preflight] Starting the kubelet service
    [certificates] Generated ca certificate and key.
    [certificates] Generated apiserver certificate and key.
    [certificates] apiserver serving cert is signed for DNS names [ubuntu-4 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.clus
    ter.local] and IPs [10.96.0.1 100.64.1.23]
    [certificates] Generated apiserver-kubelet-client certificate and key.
    [certificates] Generated sa key and public key.
    [certificates] Generated front-proxy-ca certificate and key.
    [certificates] Generated front-proxy-client certificate and key.
    [certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
    [kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
    [controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
    [controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
    [controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
    [etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
    [init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"
    [init] This often takes around a minute; or longer if the control plane images have to be pulled.
    [apiclient] All control plane components are healthy after 32.504184 seconds
    [uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [markmaster] Will mark node ubuntu-4 as master by adding a label and a taint
    [markmaster] Master ubuntu-4 tainted and labelled with key/value: node-role.kubernetes.io/master=""
    [bootstraptoken] Using token: 33794e.456bdf6538ec8783
    [bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [addons] Applied essential addon: kube-dns
    [addons] Applied essential addon: kube-proxy

    Your Kubernetes master has initialized successfully!

    To start using your cluster, you need to run (as a regular user):

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      http://kubernetes.io/docs/admin/addons/

    You can now join any number of machines by running the following on each node
    as root:

      kubeadm join --token 33794e.456bdf6538ec8783 100.64.1.23:6443 --discovery-token-ca-cert-hash sha256:518238e71fd5484de58d15d91cb7196498f23154a924bc249bc7453487246f59
    ```

* If everything is fine, the init command will give us the token that can be used to add external worker node to the cluster.

* Copy the success message above to any text editor, especially the kubeadm join parameter. We will need this to join worker node later.

* Exit from sudo. All the remaining command will be done as normal user.

    ```
    root@ubuntu-4:/home/ubuntu# exit
    exit
    ```


### Prepare the non-root user environment to be able to run kubectl command.

* Copy the config and the key to ~/.kube  

    ```
    ubuntu@ubuntu-4:~$ rm -rf .kube
    ubuntu@ubuntu-4:~$ mkdir -p $HOME/.kube
    ubuntu@ubuntu-4:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    ubuntu@ubuntu-4:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```


* Run simple sanity check, to see if all Kubernetes component is fine

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods --all-namespaces -o wide
    NAMESPACE     NAME                               READY     STATUS    RESTARTS   AGE       IP            NODE
    kube-system   etcd-ubuntu-4                      1/1       Running   0          19m       100.64.1.23   ubuntu-4
    kube-system   kube-apiserver-ubuntu-4            1/1       Running   0          19m       100.64.1.23   ubuntu-4
    kube-system   kube-controller-manager-ubuntu-4   1/1       Running   0          19m       100.64.1.23   ubuntu-4
    kube-system   kube-dns-545bc4bfd4-wnjxc          0/3       Pending   0          20m       <none>        <none>
    kube-system   kube-proxy-jm7xj                   1/1       Running   0          20m       100.64.1.23   ubuntu-4
    kube-system   kube-scheduler-ubuntu-4            1/1       Running   0          19m       100.64.1.23   ubuntu-4
    ```

* I believe it is ok for kube-dns to be pending. It will be up once we configure the networking part and have the first container created.



### Install Calico Plugin

* Download Calico manifest file in YAML format

    ```  
    wget https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
    ```

* Adjust the IP Pool range inside Calico manifest
	* By default Calico is using 192.168.0.0/16 as IP Pool for the container. To change that:
	    * edit calico.yaml
    	* find  attribute CALICO_IPV4POOL_CIDR
	    * change the value to CIDR that you want e.g: 10.201.0.0/24


* Install Calico

    ```
    ubuntu@ubuntu-4:~$ kubectl apply -f calico.yaml  
    configmap "calico-config" created
    daemonset "calico-etcd" created
    service "calico-etcd" created
    daemonset "calico-node" created
    deployment "calico-kube-controllers" created
    deployment "calico-policy-controller" created
    clusterrolebinding "calico-cni-plugin" created
    clusterrole "calico-cni-plugin" created
    serviceaccount "calico-cni-plugin" created
    clusterrolebinding "calico-kube-controllers" created
    clusterrole "calico-kube-controllers" created
    serviceaccount "calico-kube-controllers" created
    ```


* Verify if all Kubernetes and Calico components are up and running (It may take few minutes)

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods --all-namespaces -o wide
    NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE       IP            NODE
    kube-system   calico-etcd-n9tn2                          1/1       Running   0          25s       100.64.1.23   ubuntu-4
    kube-system   calico-kube-controllers-6ff88bf6d4-9n2zn   1/1       Running   0          23s       100.64.1.23   ubuntu-4
    kube-system   calico-node-hq75v                          2/2       Running   0          24s       100.64.1.23   ubuntu-4
    kube-system   etcd-ubuntu-4                              1/1       Running   0          20m       100.64.1.23   ubuntu-4
    kube-system   kube-apiserver-ubuntu-4                    1/1       Running   0          20m       100.64.1.23   ubuntu-4
    kube-system   kube-controller-manager-ubuntu-4           1/1       Running   0          20m       100.64.1.23   ubuntu-4
    kube-system   kube-dns-545bc4bfd4-wnjxc                  0/3       Pending   0          21m       <none>        <none>
    kube-system   kube-proxy-jm7xj                           1/1       Running   0          21m       100.64.1.23   ubuntu-4
    kube-system   kube-scheduler-ubuntu-4                    1/1       Running   0          20m       100.64.1.23   ubuntu-4
    ```

### (Optional) Back to control node command prompt, Enable the master node as worker 

By default, Kubernetes master node will not become a worker, means no user container will be run on master node. For this testing i want the master node to become worker too.

```
ubuntu@ubuntu-4:~$ kubectl taint nodes --all node-role.kubernetes.io/master-
node "ubuntu-4" untainted
```


### Check the node status

* command

    ```
    ubuntu@ubuntu-4:~$ kubectl get nodes  -o wide
    NAME       STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    ubuntu-4   Ready     master    26m       v1.8.1    <none>        Ubuntu 16.04.3 LTS   4.4.0-97-generic   docker://1.12.6
    ```

* This time we only have one worker node which in this case, master node is also the worker. 

  
  
### Install Calicoctl

Calicoctl is a utility to manage Calico plugin.

* To install, run:

    ```
    ubuntu@ubuntu-4:~$ kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/calicoctl.yaml
    pod "calicoctl" created
    ```

* Check the calicoctl installation status

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods --all-namespaces -o wide
    NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE       IP             NODE
    ... deleted...

    kube-system   calicoctl                                  1/1       Running   0          6s        100.64.1.23    ubuntu-4

    ... deleted...
    ubuntu@ubuntu-4:~$
    ```

* Check if Calicoctl works  

    ```  
    ubuntu@ubuntu-4:~$ kubectl exec -ti -n kube-system calicoctl -- /calicoctl get profiles -o wide
    NAME                 TAGS
    k8s_ns.default
    k8s_ns.kube-public
    k8s_ns.kube-system
    ```


## Run the Containers

### Bring up some basic containers for sanity check

* Bring up a single ubuntu-based sshd container

    ```
    ubuntu@ubuntu-4:~$ kubectl run sshd-1 --image=rastasheep/ubuntu-sshd:16.04
    deployment "sshd-1" created
    ```


* Check the container status

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods -o wide
    NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
    sshd-1-84c4bf4558-284dj   1/1       Running   0          23s       10.201.0.197   ubuntu-4

    ubuntu@ubuntu-4:~$ kubectl describe pod sshd-1-84c4bf4558-284dj
    Name:           sshd-1-84c4bf4558-284dj
    Namespace:      default
    Node:           ubuntu-4/100.64.1.23
    Start Time:     Thu, 19 Oct 2017 18:25:33 +0000
    Labels:         pod-template-hash=4070690114
                    run=sshd-1
    Annotations:    kubernetes.io/created-by={"kind":"SerializedReference","apiVersion":"v1","reference":{"kind":"ReplicaSet","namespace":"default","name":"sshd-1-84c4bf4558","uid":"e4bffd65-b4fa-11e7-b2dc-02f5832b70b6",...
    Status:         Running
    IP:             10.201.0.197
    Created By:     ReplicaSet/sshd-1-84c4bf4558
    Controlled By:  ReplicaSet/sshd-1-84c4bf4558
    Containers:
      sshd-1:
        Container ID:   docker://ea18ceef6208555a319c715a2f2170643d4cac56eff53b747a2d1026582ff55e
        Image:          rastasheep/ubuntu-sshd:16.04
        Image ID:       docker-pullable://rastasheep/ubuntu-sshd@sha256:62c2957f0af93817ee6d88da4cbe41ee1b318b39eb3394a3571dffa2585ad290
        Port:           <none>
        State:          Running
          Started:      Thu, 19 Oct 2017 18:25:35 +0000
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-h52zb (ro)
    Conditions:
      Type           Status
      Initialized    True 
      Ready          True 
      PodScheduled   True 
    Volumes:
      default-token-h52zb:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-h52zb
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.alpha.kubernetes.io/notReady:NoExecute for 300s
                     node.alpha.kubernetes.io/unreachable:NoExecute for 300s
    Events:          <none>
    ```


## Join the second worker node to the cluster 

* In this example, i am going to join ubuntu-3 host as a second worker node.

* We will use kubeadm join command and the key that was provided after we finished with kubeadm init.

    ```
    root@ubuntu-3:/home/ubuntu# kubeadm join --token 33794e.456bdf6538ec8783 100.64.1.23:6443 --discovery-token-ca-cert-hash sha256:518238e71fd5484de58d15d91cb7196498f23154a924bc249bc7453487246f59
    [kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
    [preflight] Running pre-flight checks
    [discovery] Trying to connect to API Server "100.64.1.23:6443"
    [discovery] Created cluster-info discovery client, requesting info from "https://100.64.1.23:6443"
    [discovery] Requesting info from "https://100.64.1.23:6443" again to validate TLS against the pinned public key
    [discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "100.64.1.23:6443"
    [discovery] Successfully established connection with API Server "100.64.1.23:6443"
    [bootstrap] Detected server version: v1.8.1
    [bootstrap] The server supports the Certificates API (certificates.k8s.io/v1beta1)

    Node join complete:
    * Certificate signing request sent to master and response
      received.
    * Kubelet informed of new secure connection details.

    Run 'kubectl get nodes' on the master to see this machine join.
    ```

* Verify the node list

    ```
    ubuntu@ubuntu-4:~$ kubectl get nodes -o wide
    NAME       STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    ubuntu-3   Ready     <none>    38s       v1.8.1    <none>        Ubuntu 16.04.3 LTS   4.4.0-97-generic   docker://1.12.6
    ubuntu-4   Ready     master    50m       v1.8.1    <none>        Ubuntu 16.04.3 LTS   4.4.0-97-generic   docker://1.12.6
    ```


* Verify the system containers again

    ```
    ubuntu@ubuntu-4:~$ kubectl get pods --all-namespaces -o wide
    NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE       IP             NODE
    default       sshd-1-84c4bf4558-284dj                    1/1       Running   0          22m       10.201.0.197   ubuntu-4
    kube-system   calico-etcd-n9tn2                          1/1       Running   0          29m       100.64.1.23    ubuntu-4
    kube-system   calico-kube-controllers-6ff88bf6d4-9n2zn   1/1       Running   0          29m       100.64.1.23    ubuntu-4
    kube-system   calico-node-hq75v                          2/2       Running   0          29m       100.64.1.23    ubuntu-4
    kube-system   calico-node-xgf66                          1/2       Running   1          51s       100.64.1.24    ubuntu-3
    kube-system   calicoctl                                  1/1       Running   0          24m       100.64.1.23    ubuntu-4
    kube-system   etcd-ubuntu-4                              1/1       Running   0          49m       100.64.1.23    ubuntu-4
    kube-system   kube-apiserver-ubuntu-4                    1/1       Running   0          49m       100.64.1.23    ubuntu-4
    kube-system   kube-controller-manager-ubuntu-4           1/1       Running   0          49m       100.64.1.23    ubuntu-4
    kube-system   kube-dns-545bc4bfd4-wnjxc                  3/3       Running   0          50m       10.201.0.196   ubuntu-4
    kube-system   kube-proxy-jm7xj                           1/1       Running   0          50m       100.64.1.23    ubuntu-4
    kube-system   kube-proxy-t82nz                           1/1       Running   0          51s       100.64.1.24    ubuntu-3
    kube-system   kube-scheduler-ubuntu-4                    1/1       Running   0          49m       100.64.1.23    ubuntu-4
    ```

* From the output above, we can see that 2 system containers were installed on worker node. Those containers are kube-proxy and calico-node. 


If everything looks OK, let's check the setup in more detail. I'll cover this in the next post [Calico and Kubernetes part 2](2017-10-20-Calico-and-Kubernetes-part-2.md)




### MISC

* As a side node, kubeadm init, for some reason does not accept prefix smaller than /24. 
    * I was curious about what is the smallest prefix to run kubeadm init, so i tried to run init with /25 prefix, and one of the container keep crashing
    * Sample output
    
        ```
        ubuntu@ubuntu-4:~$ kubectl get pods --all-namespaces -o wide
        NAMESPACE     NAME                               READY     STATUS             RESTARTS   AGE       IP            NODE
        kube-system   etcd-ubuntu-4                      1/1       Running            0          17s       100.64.1.23   ubuntu-4
        kube-system   kube-apiserver-ubuntu-4            1/1       Running            0          9s        100.64.1.23   ubuntu-4
        kube-system   kube-controller-manager-ubuntu-4   0/1       CrashLoopBackOff   3          1m        100.64.1.23   ubuntu-4
        kube-system   kube-scheduler-ubuntu-4            1/1       Running            0          11s       100.64.1.23   ubuntu-4    
        ```





## Links
* [Kubernetes and Calico Part 2](2017-10-20-Calico-and-Kubernetes-part-2.md)
* [Kubernetes and Calico Part 2](2017-10-21-Calico-and-Kubernetes-part-3.md)
