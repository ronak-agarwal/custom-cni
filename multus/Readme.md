## Pod with Two NICs

1. Add two interfaces in VMs (enable two interfaces in Vbox)

2. kubectl apply -f multus.yaml (configure custom CNI as default)

3. kubectl apply -f macvlan.yaml (configure according to 2nd NIC of VMs)

4. Finally create pod using annotation to add dual NICs

To test run exec in pod

```hcl
[root@master ~]# kubectl exec samplepod -- ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: net1@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether 4a:b7:30:6a:e1:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.230.0.100/24 brd 10.230.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fd17:625c:f037:e600:48b7:30ff:fe6a:e1d3/64 scope global dynamic 
       valid_lft forever preferred_lft forever
    inet6 fe80::48b7:30ff:fe6a:e1d3/64 scope link 
       valid_lft forever preferred_lft forever
11: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 92:6b:f2:b7:b6:fd brd ff:ff:ff:ff:ff:ff
    inet 10.240.1.15/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::906b:f2ff:feb7:b6fd/64 scope link 
       valid_lft forever preferred_lft forever

```