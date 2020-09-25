# custom-cni

You can write your custom CNI plugin for k8s

## Steps to setup

1. Create k8s 2 node cluster (using Centos7 minimal), you can use kubeadm init.

2. Install jq and bridge-utils on centos

yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install jq

yum -y install bridge-utils

3. Copy custom-cni to /opt/cni/bin/custom-cni


Typically any CNI script has ADD / DEL verbs

Script contains -

a) Create a bridge on node

b) Script to generate IP from POD CIDR range provided in kubeadm

c) Create veth on host (node) which will be mapped to pod eth0

d) Assign above veth nic to network namespace created by kubelet (using Docker runtime)

e) Assign IP address to network namespace (pod) this is IPAM


You can see CNI logs as configured in the script at this place => /var/log/cni.log

NOTE - There is a custom script to generate IP of POD (acting as IPAM), which stores a file in /tmp/ and increment by +1 everytime pod is created

### Below steps are repeated for every node

4. Copy 10-custom-cni.conf to /etc/cni/net.d/10-custom-cni.conf

Change pod cidr range on every node (Eg Node1 = 10.240.0.0/24 , Node2 = 10.240.1.0/24)

[![CNI-config.png](https://github.com/ronak-agarwal/custom-cni/blob/master/images/CNI-config.png)]()

5. Initiate k8s cluster setup on master using kubeadm

kubeadm init --pod-network-cidr=10.240.0.0/24 (Note this Pod CIDR range is for entire cluster and should cover node specific pod CIDR range configured in 10-custom-cni.conf file /24 range )

So below are pod CIDR ranges configured -
```hcl
Node1 (10.240.1.0/24)
Node2 (10.240.0.0/24)
```

### Before trying below deploy 3 pods

```hcl
kubectl apply -f pods.yaml

two pods will land on Node2 (alpine and nginx1)
one pod will land on Node1 (nginx2)
```

6. ADD static iptable rules to enable Pod to Pod communication (on same host)

Eg - Add On Node2

```hcl
iptables -A FORWARD -s 10.240.0.0/24 -j ACCEPT  (/24 entire node pod cidr)

iptables -A FORWARD -d 10.240.0.0/24 -j ACCEPT
```

Forward chain is responsible for allowing such chain

kubectl exec alpine1 -- wget -qO- 10.240.0.8 (Pod to Pod on same node, assuming alpine and nginx1 is running on Node2)

Similarly add on Node1
```hcl
iptables -A FORWARD -s 10.240.1.0/24 -j ACCEPT  (/24 entire node pod cidr)

iptables -A FORWARD -d 10.240.1.0/24 -j ACCEPT
```

7. ADD route to Allow communication across hosts
```hcl
ip route add 10.240.1.0/24 via 10.0.2.14 dev enp0s3 (add in Node2)
```
Route any packet for node1 podcidr (10.240.1.0/24) to node1 ip via device enp0

```hcl
ip route add 10.240.0.0/24 via 10.0.2.15 dev enp0s3 (add in Node1)
```
Route any packet for node2 podcidr (10.240.0.0/24) to node2 ip via device enp0


8. Allow outgoing internet from Pods by adding NAT rule in iptables

On Node2
```hcl
iptables -t nat -A POSTROUTING -s 10.240.0.0/24 ! -o cni0 -j MASQUERADE
```
Add in POSTROUTING chain (last chain) to evaluate outgoing packet, and need to add in linux NAT table and making sure only those packet which are not going out to cni bridge

MASQUERADE meaning apply to source net (replace source IP with host IP)

kubectl exec alpine1 -- ping 8.8.8.8 (Assuming alpine pod running on node2)

## Other links

I did similar work here which is confined to only container network - https://github.com/ronak-agarwal/rocker

Setup k8s on Centos7 - https://medium.com/@genekuo/setting-up-a-multi-node-kubernetes-cluster-on-a-laptop-69ae3e3d0f7c
