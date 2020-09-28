# custom-cni

You can write your custom CNI plugin for k8s -

Every CNI has usually a binary and a daemon, binary - create Pod NIC and act as IPAM, where as daemon - adds routing / iptables rules on host to manage pod-pod communication


## Prerequisite

Create VM snapshot (using Centos7 minimal) which has below binaries and configs.

```hcl
Install jq and bridge-utils on centos

yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install jq

yum -y install bridge-utils wget

systemctl disable firewalld && systemctl stop firewalld
/etc/selinux/config
SELINUX=permissive

Configure repo

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum install -y docker kubelet kubeadm kubectl kubenetes-cni

systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet

sysctl -w net.bridge.bridge-nf-call-iptables=1
echo "net.bridge.bridge-nf-call-iptables=1" > /etc/sysctl.d/k8s.conf
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab
```

### Below steps are repeated for every node

1. Copy custom-cni to /opt/cni/bin/custom-cni

Typically any CNI script has ADD / DEL verbs

Below script is to create veth on host for every pod and assign Pod IPs based on Pod CIDR range of each node and link that to a virtual bridge

Script contains -

a) Create a bridge on node

b) Script to generate IP from POD CIDR range provided in kubeadm

c) Create veth on host (node) which will be mapped to pod eth0

d) Assign above veth nic to network namespace created by kubelet (using Docker runtime)

e) Assign IP address to network namespace (pod) this is IPAM


You can see CNI logs as configured in the script at this place => /var/log/cni.log

NOTE - There is a custom script to generate IP of POD (acting as IPAM), which stores a file in /tmp/ and increment by +1 everytime pod is created



2. Copy 10-custom-cni.conf to /etc/cni/net.d/10-custom-cni.conf

Change pod cidr range on every node (Eg Node1 = 10.240.0.0/24 , Node2 = 10.240.1.0/24)

[![CNI-config.png](https://github.com/ronak-agarwal/custom-cni/blob/master/images/CNI-config.png)]()

3. Initiate k8s cluster setup on master using kubeadm

kubeadm init --pod-network-cidr=10.240.0.0/24 (Note this Pod CIDR range is for entire cluster and should cover node specific pod CIDR range configured in 10-custom-cni.conf file /24 range )

So below are pod CIDR ranges configured -
```hcl
Node1 (10.240.1.0/24)
Node2 (10.240.0.0/24)
```

[![Setup.png](https://github.com/ronak-agarwal/custom-cni/blob/master/images/Setup.png)]()

### Packet Flow Between Pods

This is to show packet flow (actual data) from Pod to Pod (on same host and different hosts)

```hcl
kubectl apply -f pods.yaml

two pods will land on Node2 (alpine and nginx1)
one pod will land on Node1 (nginx2)
```

4. ADD static iptable rules to enable Pod to Pod communication (on same host)

[![SameHost.png](https://github.com/ronak-agarwal/custom-cni/blob/master/images/SameHost.png)]()

Eg - Add On Node2

```hcl
eth0 (in Pod A’s netns) → vethA → br0 → vethB → eth0 (in Pod B’s netns)

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

5. ADD route to Allow communication across hosts

[![DifferentHost.png](https://github.com/ronak-agarwal/custom-cni/blob/master/images/DifferentHost.png)]()

```hcl
Ideal flow from CNI plugin, since I don't have vxlan (IPIP encapsulation) so I have used source nat (MASQUERADE) which does same thing  
eth0 (in Pod A’s netns) → vethA → br0 → vxlan0 → physical network [underlay] → vxlan0 → br0 → vethB → eth0 (in Pod C’s netns)

ip route add 10.240.1.0/24 via 10.0.2.14 dev enp0s3 (add in Node2)
```
Route any packet for node1 podcidr (10.240.1.0/24) to node1 ip via device enp0

```hcl
ip route add 10.240.0.0/24 via 10.0.2.15 dev enp0s3 (add in Node1)
```
Route any packet for node2 podcidr (10.240.0.0/24) to node2 ip via device enp0


6. Allow outgoing internet from Pods by adding NAT rule in iptables

On Node2
```hcl
Below is the flow from Pod A to get Internet, so I have added POSTROUTING nat rule
eth0 (in Pod A’s netns) → vethA → br0 → (NAT) → eth0 (physical device) → Internet

iptables -t nat -A POSTROUTING -s 10.240.0.0/24 ! -o cni0 -j MASQUERADE
```
Add in POSTROUTING chain (last chain) to evaluate outgoing packet, and need to add in linux NAT table and making sure only those packet which are not going out to cni bridge

MASQUERADE meaning apply to source nat (replace source IP with host IP)

kubectl exec alpine1 -- ping 8.8.8.8 (Assuming alpine pod running on node2)

## K8S Services

Kubernetes Services are not part of CNI plugin and managed natively via kube-proxy so if Pod to Pod communication sorted then Services > Pod communication just work straight away, below Service IP will be reachable from both Nodes (Hosts / Pods)

```hcl
kubectl label pods nginx2 app=nginx2
kubectl expose pod nginx2 --name=nginx2 --port=8000 --target-port=80
```

### Some commands

```hcl
iptables -S FORWARD (to see FORWARD chain)
iptables -t nat -L KUBE-SERVICES (to see kube-proxy -s and -d nating)

Using tshark to see source and destination of packet along with protocol
tshark -i cni0 -T fields -e ip.src -e ip.dst -e frame.protocols -E header=y
```

## Other links

I did similar work here which is confined to only container network - https://github.com/ronak-agarwal/rocker

Setup k8s on Centos7 - https://medium.com/@genekuo/setting-up-a-multi-node-kubernetes-cluster-on-a-laptop-69ae3e3d0f7c

https://github.com/kristenjacobs/container-networking
