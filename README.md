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

4. Copy 10-custom-cni.conf to /etc/cni/net.d/10-custom-cni.conf

[![CNI-config.png](https://github.com/ronak-agarwal/custom-cni/blob/master/images/CNI-config.png)]()

5. Initiate k8s cluster setup on master using kubeadm

kubeadm init --pod-network-cidr=10.240.0.0/24 (Note this CIDR range is same which is configured in conf file /24 range )



6. ADD static iptable rules to enable Pod to Pod communication (even on same host)

iptables -A FORWARD -s 10.240.0.0/16 -j ACCEPT
iptables -A FORWARD -d 10.240.0.0/16 -j ACCEPT

7. ADD route to Allow communication across hosts

ip route add 10.240.1.0/24 via 10.10.10.11 dev enp0s3

8. Allow outgoing internet from Pods by adding NAT rule in iptables

iptables -t nat -A POSTROUTING -s 10.240.0.0/24 ! -o cni0 -j MASQUERADE


## Other links

I did a similar work here which is confined to only container network - https://github.com/ronak-agarwal/rocker
