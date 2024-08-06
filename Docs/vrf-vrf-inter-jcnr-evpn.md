# VRF - EVPN/VXLAN : JCNR間Routing
- VRF Instance Type : vrf
- JCNR WorkerNode間でPOD間VXLANによるL3接続
- POD / JCNR vRouter間はL3接続

## VRF EVPN/VXLAN - JCNR間Routing
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-evpn-vxlan.png" width=800>

## JCNR Configlet
[JCNR1 Configlet Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/configlet-evpnvxlan-jcnr1.yaml)

[JCNR2 Configlet Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/configlet-evpnvxlan-jcnr2.yaml)

### Network Attachment Definition　作成
[VRF NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-nad-jcnr1.yaml)

[VRF NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-nad-jcnr2.yaml)

#### NAD Option
- InstanceType : VRF Liteは"vrf"を使用
- vrfTarget : VRF間接続時はvrfTargetを揃える
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
- NADで指定したRouteがStatic Routeとして反映される
```
[root@jcnr1]# kubectl describe pod vrf1-pod1
Name:         vrf1-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Mon, 16 Oct 2023 01:35:49 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 9b6389c9e5c1a7b7f348d226f07d6fc1b90cee58e4c827ce2a5edcf34321f409
              cni.projectcalico.org/podIP: 172.30.79.18/32
              cni.projectcalico.org/podIPs: 172.30.79.18/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.18"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.53",
                        "abcd::a00:12"
                    ],
                    "mac": "02:00:00:87:A8:56",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrf1",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth",
                      "vrfTarget": "64512:1"
                    }
                  }
                ]
Status:       Running
--- snip ---
```
```
[root@jcnr2]# kubectl describe pod vrf1-pod2
Name:         vrf1-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Mon, 16 Oct 2023 01:36:49 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: b1ede4237388801f8f8e83250038d0d98cca89d488c289bf727bbf3e69f403c0
              cni.projectcalico.org/podIP: 172.30.147.253/32
              cni.projectcalico.org/podIPs: 172.30.147.253/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.253"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "20.0.0.5",
                        "abcd::1400:5"
                    ],
                    "mac": "02:00:00:A5:88:77",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrf1",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth",
                      "vrfTarget": "64512:1"
                    }
                  }
                ]
Status:       Running
--- snip ---
```

POD IP/Route 確認
```
[root@vrf1-pod1 /]# ip a
--- snip ---
413: net1@if414: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:87:a8:56 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.53/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:12/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe87:a856/64 scope link
       valid_lft forever preferred_lft forever

[root@vrf1-pod1 /]# ip route
default via 10.0.0.1 dev net1
10.0.0.0/24 via 10.0.0.1 dev net1
10.0.0.1 dev net1 scope link
20.0.0.0/24 via 10.0.0.1 dev net1
169.254.1.1 dev eth0 scope link

[root@vrf1-pod1 /]# ip -6 route
abcd::a00:1 dev net1 metric 1024 pref medium
abcd::a00:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
abcd::1400:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev net1 proto kernel metric 256 pref medium
```
```
[root@vrf1-pod2 /]# ip a
--- snip ---
186: net1@if187: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:a5:88:77 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.5/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:5/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fea5:8877/64 scope link
       valid_lft forever preferred_lft forever

[root@vrf1-pod2 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.1 dev net1 scope link
169.254.1.1 dev eth0 scope link

[root@vrf1-pod2 /]# ip -6 route
abcd::a00:0/120 via abcd::1400:1 dev net1 metric 1024 pref medium
abcd::1400:1 dev net1 metric 1024 pref medium
abcd::1400:0/120 via abcd::1400:1 dev net1 metric 1024 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev net1 proto kernel metric 256 pref medium
```

cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni routing-instances vrf1 instance-type vrf
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::a00:12/128 qualified-next-hop abcd::a00:12 interface jvknet1-e5a495e
set groups cni routing-instances vrf1 routing-options static route 10.0.0.53/32 qualified-next-hop 10.0.0.53 interface jvknet1-e5a495e
set groups cni routing-instances vrf1 interface jvknet1-e5a495e
set groups cni routing-instances vrf1 vrf-target target:64512:1
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vrf1 instance-type vrf
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::1400:5/128 qualified-next-hop abcd::1400:5 interface jvknet1-b12d2e9
set groups cni routing-instances vrf1 routing-options static route 20.0.0.5/32 qualified-next-hop 20.0.0.5 interface jvknet1-b12d2e9
set groups cni routing-instances vrf1 interface jvknet1-b12d2e9
set groups cni routing-instances vrf1 vrf-target target:64512:1
```

cRPD Route確認
```
root@jcnr1> show bgp summary
Threading mode: BGP I/O
TCP listen port: 178
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.2               64512         19         18       0       1        7:01 Establ
  bgp.evpn.0: 2/2/2/0
  vrf1.evpn.0: 2/2/2/0

root@jcnr1> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 00:33:41
                    >  via jvknet1-e5a495e
10.0.0.53/32       *[Local/0] 00:33:41
                       Local via jvknet1-e5a495e
20.0.0.0/24        *[EVPN/170] 00:07:24
                    >  to 192.168.0.2 via ens4

root@jcnr1> show route table vrf1.evpn.0

vrf1.evpn.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:1.1.1.1:3::0::10.0.0.0::24/248
                   *[EVPN/170] 00:27:55
                       Fictitious
5:1.1.1.2:3::0::20.0.0.0::24/248
                   *[BGP/170] 00:07:38, localpref 100, from 1.1.1.2
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.2 via ens4, Push 2000
5:1.1.1.1:3::0::abcd::a00:0::120/248
                   *[EVPN/170] 00:27:55
                       Fictitious
5:1.1.1.2:3::0::abcd::1400:0::120/248
                   *[BGP/170] 00:07:38, localpref 100, from 1.1.1.2
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.2 via ens4, Push 2000
```
```
root@jcnr2> show bgp summary
Threading mode: BGP I/O
TCP listen port: 178
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.1               64512         22         20       0       0        8:17 Establ
  bgp.evpn.0: 2/2/2/0
  vrf1.evpn.0: 2/2/2/0

root@jcnr2> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[EVPN/170] 00:08:25
                    >  to 192.168.0.1 via ens4
20.0.0.0/24        *[Direct/0] 00:37:06
                    >  via jvknet1-b12d2e9
20.0.0.5/32        *[Local/0] 00:37:06
                       Local via jvknet1-b12d2e9

root@jcnr2> show route table vrf1.evpn.0

vrf1.evpn.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:1.1.1.1:3::0::10.0.0.0::24/248
                   *[BGP/170] 00:08:28, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.1 via ens4, Push 1000
5:1.1.1.2:3::0::20.0.0.0::24/248
                   *[EVPN/170] 00:27:38
                       Fictitious
5:1.1.1.1:3::0::abcd::a00:0::120/248
                   *[BGP/170] 00:08:28, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.1 via ens4, Push 1000
5:1.1.1.2:3::0::abcd::1400:0::120/248
                   *[EVPN/170] 00:27:38
                       Fictitious
```


vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-dpjmx -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:7 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:7  bytes:646 errors:0
            TX packets:15  bytes:1526 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:2919  bytes:270511 errors:0
            RX port   packets:2919 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:2919  bytes:270511 errors:0
            TX packets:2147  bytes:197631 errors:0
            Drops:0
            TX queue  packets:2147 errors:0
            TX port   packets:2147 errors:0
            TX device packets:2156  bytes:198633 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:7  bytes:826 errors:0
            RX port   packets:4 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:7  bytes:842 errors:3
            TX packets:0  bytes:0 errors:0
            Drops:4

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:1
            RX device packets:2140  bytes:197293 errors:0
            RX queue  packets:2140 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:2140  bytes:197293 errors:0
            TX packets:2919  bytes:270511 errors:0
            Drops:0
            TX queue  packets:2919 errors:0
            TX device packets:2919  bytes:270511 errors:0

vif0/6      Ethernet: jvknet1-e5a495e NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:10.0.0.53
            IP6addr:abcd::a00:12
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:55 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:55  bytes:11938 errors:0
            TX packets:3  bytes:258 errors:0
            Drops:52
            TX port   packets:3 errors:0
```
```
[root@jcnr2 ~]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:9 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:9  bytes:818 errors:0
            TX packets:14  bytes:1328 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:2141  bytes:197178 errors:0
            RX port   packets:2138 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:2138  bytes:196911 errors:0
            TX packets:2933  bytes:271666 errors:0
            Drops:0
            TX queue  packets:2933 errors:0
            TX port   packets:2933 errors:0
            TX device packets:2941  bytes:272578 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:ac:04:91
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:1
            RX device packets:2924  bytes:271244 errors:0
            RX queue  packets:2924 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:2924  bytes:271244 errors:0
            TX packets:2138  bytes:196911 errors:0
            Drops:0
            TX queue  packets:2138 errors:0
            TX device packets:2138  bytes:196911 errors:0

vif0/6      Ethernet: jvknet1-b12d2e9 NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:20.0.0.5
            IP6addr:abcd::1400:5
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:94 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:94  bytes:17949 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:94
```

POD間疎通確認
```
[root@vrf1-pod1 /]# ping 20.0.0.5
PING 20.0.0.5 (20.0.0.5) 56(84) bytes of data.
64 bytes from 20.0.0.5: icmp_seq=1 ttl=62 time=1.59 ms
64 bytes from 20.0.0.5: icmp_seq=2 ttl=62 time=0.332 ms
64 bytes from 20.0.0.5: icmp_seq=3 ttl=62 time=0.331 ms
```
