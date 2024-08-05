# VRF - VRF L3VPN / SR-MPLS : JCNR間Routing
- VRF Instance Type : vrf
- JCNR WorkerNode間でPOD間SR-MPLSによるL3接続
- POD / JCNR vRouter間はL3接続

## VRF L3VPN / SR-MPLS - JCNR間Routing
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-srmpls.png" width=800>

### Configlet
[JCNR1 Configlet Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/configlet-srmpls-jcnr1.yaml)
[JCNR2 Configlet Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/configlet-srmpls-jcnr2.yaml)


### Network Attachment Definition　作成
[VRF L3VPN NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-nad-jcnr1.yaml)

[VRF L3VPN NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-nad-jcnr2.yaml)

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
Start Time:   Mon, 16 Oct 2023 21:16:17 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: f673217b111eb2b9da68d21826f45a6d4221797fb4947af7cba578591067a0fd
              cni.projectcalico.org/podIP: 172.30.79.19/32
              cni.projectcalico.org/podIPs: 172.30.79.19/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.19"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.54",
                        "abcd::a00:13"
                    ],
                    "mac": "02:00:00:05:19:07",
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
Start Time:   Mon, 16 Oct 2023 21:16:41 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 12b9b29c936f5e090eea131a005e2248610bbca4692efe47cffae6e6ef67ff0e
              cni.projectcalico.org/podIP: 172.30.147.254/32
              cni.projectcalico.org/podIPs: 172.30.147.254/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.254"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "20.0.0.6",
                        "abcd::1400:6"
                    ],
                    "mac": "02:00:00:AA:75:CA",
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
423: net1@if424: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:05:19:07 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.54/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:13/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe05:1907/64 scope link
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
196: net1@if197: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:aa:75:ca brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.6/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:6/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:feaa:75ca/64 scope link
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
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::a00:13/128 qualified-next-hop abcd::a00:13 interface jvknet1-1229f7b
set groups cni routing-instances vrf1 routing-options static route 10.0.0.54/32 qualified-next-hop 10.0.0.54 interface jvknet1-1229f7b
set groups cni routing-instances vrf1 interface jvknet1-1229f7b
set groups cni routing-instances vrf1 vrf-target target:64512:1
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vrf1 instance-type vrf
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::1400:6/128 qualified-next-hop abcd::1400:6 interface jvknet1-808ed81
set groups cni routing-instances vrf1 routing-options static route 20.0.0.6/32 qualified-next-hop 20.0.0.6 interface jvknet1-808ed81
set groups cni routing-instances vrf1 interface jvknet1-808ed81
set groups cni routing-instances vrf1 vrf-target target:64512:1
```

cRPD Route確認
```
root@jcnr1> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ens4                  jcnr2          2  Up                     7  52:54:0:cf:74:89

root@jcnr1> show isis database detail
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

jcnr1.00-00 Sequence: 0x74, Checksum: 0x897b, Lifetime: 669 secs
  IPV4 Index: 2001
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      23, Weight:   0, Neighbor: jcnr2, Flags: --VL--
     LAN IPv6 Adj-SID:      24, Weight:   0, Neighbor: jcnr2, Flags: F-VL--
   IP prefix: 1.1.1.1/32                      Metric:        0 Internal Up
   IP prefix: 169.254.119.159/32              Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::8f2:20ff:fee5:be2d/128    Metric:        0 Internal Up

jcnr2.00-00 Sequence: 0x71, Checksum: 0xc363, Lifetime: 387 secs
  IPV4 Index: 2002
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      16, Weight:   0, Neighbor: jcnr1, Flags: --VL--
     LAN IPv6 Adj-SID:      17, Weight:   0, Neighbor: jcnr1, Flags: F-VL--
   IP prefix: 1.1.1.2/32                      Metric:        0 Internal Up
   IP prefix: 169.254.44.237/32               Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::b811:f3ff:fe92:a8b9/128   Metric:        0 Internal Up

jcnr2.02-00 Sequence: 0x6c, Checksum: 0xb538, Lifetime: 871 secs
   IS neighbor: jcnr1.00                      Metric:        0
   IS neighbor: jcnr2.00                      Metric:        0

root@jcnr1> show route table mpls.0

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 23:44:09, metric 1
                       Receive
1                  *[MPLS/0] 23:44:09, metric 1
                       Receive
2                  *[MPLS/0] 23:44:09, metric 1
                       Receive
13                 *[MPLS/0] 23:44:09, metric 1
                       Receive
23                 *[L-ISIS/14] 04:04:47, metric 0
                    >  to 192.168.0.2 via ens4, Pop
23(S=0)            *[L-ISIS/14] 04:04:47, metric 0
                    >  to 192.168.0.2 via ens4, Pop
24                 *[L-ISIS/14] 04:04:47, metric 0
                    >  to fe80::5054:ff:fecf:7489 via ens4, Pop
24(S=0)            *[L-ISIS/14] 04:04:47, metric 0
                    >  to fe80::5054:ff:fecf:7489 via ens4, Pop
25                 *[VPN/170] 04:04:21
                    >  via jvknet1-1229f7b, Pop
25(S=0)            *[VPN/170] 04:04:21
                    >  via jvknet1-1229f7b, Pop
26                 *[VPN/170] 04:04:21
                    >  via jvknet1-1229f7b, Pop
26(S=0)            *[VPN/170] 04:04:21
                    >  via jvknet1-1229f7b, Pop
402002             *[L-ISIS/14] 04:04:47, metric 10
                    >  to 192.168.0.2 via ens4, Pop
402002(S=0)        *[L-ISIS/14] 04:04:47, metric 10
                    >  to 192.168.0.2 via ens4, Pop

root@jcnr1> show bgp summary
Threading mode: BGP I/O
TCP listen port: 178
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l3vpn.0
                       1          1          0          0          0          0
bgp.l3vpn-inet6.0
                       1          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.2               64512         65         64       0       1       26:34 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/1/1/0
  vrf1.inet.0: 1/1/1/0
  vrf1.inet6.0: 0/1/1/0

root@jcnr1> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 01:19:55
                    >  via jvknet1-1229f7b
10.0.0.54/32       *[Local/0] 01:19:55
                       Local via jvknet1-1229f7b
20.0.0.0/24        *[BGP/170] 00:26:51, localpref 100, from 1.1.1.2
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.2 via ens4, Push 18

root@jcnr1> show route table mpls.0 label 25 detail

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
25  (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x563bad42927c
                Next-hop reference count: 3, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-1229f7b, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x563bb021c128
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 4:10:05
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (3): 1-KRT MFS 2-KRT 4-KRT-vRouter
                AS path: I
		Ref Cnt: 1
                Thread: junos-main

25(S=0) (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x563bad42d5dc
                Next-hop reference count: 1, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-1229f7b, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x563bb021bee8
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 4:10:05
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (1): 1-KRT MFS
                AS path: I
		Ref Cnt: 1
                Thread: junos-main
```
```
root@jcnr2> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ens4                  jcnr1          2  Up                    19  52:54:0:43:f9:a1

root@jcnr2> show isis database detail
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

jcnr1.00-00 Sequence: 0x74, Checksum: 0x897b, Lifetime: 460 secs
  IPV4 Index: 2001
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      23, Weight:   0, Neighbor: jcnr2, Flags: --VL--
     LAN IPv6 Adj-SID:      24, Weight:   0, Neighbor: jcnr2, Flags: F-VL--
   IP prefix: 1.1.1.1/32                      Metric:        0 Internal Up
   IP prefix: 169.254.119.159/32              Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::8f2:20ff:fee5:be2d/128    Metric:        0 Internal Up

jcnr2.00-00 Sequence: 0x72, Checksum: 0xc164, Lifetime: 1037 secs
  IPV4 Index: 2002
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      16, Weight:   0, Neighbor: jcnr1, Flags: --VL--
     LAN IPv6 Adj-SID:      17, Weight:   0, Neighbor: jcnr1, Flags: F-VL--
   IP prefix: 1.1.1.2/32                      Metric:        0 Internal Up
   IP prefix: 169.254.44.237/32               Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::b811:f3ff:fe92:a8b9/128   Metric:        0 Internal Up

jcnr2.02-00 Sequence: 0x6c, Checksum: 0xb538, Lifetime: 665 secs
   IS neighbor: jcnr1.00                      Metric:        0
   IS neighbor: jcnr2.00                      Metric:        0

root@jcnr2> show route table mpls.0

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 23:46:43, metric 1
                       Receive
1                  *[MPLS/0] 23:46:43, metric 1
                       Receive
2                  *[MPLS/0] 23:46:43, metric 1
                       Receive
13                 *[MPLS/0] 23:46:43, metric 1
                       Receive
16                 *[L-ISIS/14] 04:07:47, metric 0
                    >  to 192.168.0.1 via ens4, Pop
16(S=0)            *[L-ISIS/14] 04:07:47, metric 0
                    >  to 192.168.0.1 via ens4, Pop
17                 *[L-ISIS/14] 04:07:47, metric 0
                    >  to fe80::5054:ff:fe43:f9a1 via ens4, Pop
17(S=0)            *[L-ISIS/14] 04:07:47, metric 0
                    >  to fe80::5054:ff:fe43:f9a1 via ens4, Pop
18                 *[VPN/170] 04:07:21
                    >  via jvknet1-808ed81, Pop
18(S=0)            *[VPN/170] 04:07:21
                    >  via jvknet1-808ed81, Pop
19                 *[VPN/170] 04:07:21
                    >  via jvknet1-808ed81, Pop
19(S=0)            *[VPN/170] 04:07:21
                    >  via jvknet1-808ed81, Pop
402001             *[L-ISIS/14] 04:07:47, metric 10
                    >  to 192.168.0.1 via ens4, Pop
402001(S=0)        *[L-ISIS/14] 04:07:47, metric 10
                    >  to 192.168.0.1 via ens4, Pop

root@jcnr2> show bgp summary
Threading mode: BGP I/O
TCP listen port: 178
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l3vpn.0
                       1          1          0          0          0          0
bgp.l3vpn-inet6.0
                       1          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.1               64512        555        553       0       1     4:06:14 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/1/1/0
  vrf1.inet.0: 1/1/1/0
  vrf1.inet6.0: 0/1/1/0

root@jcnr2> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[BGP/170] 04:06:30, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.1 via ens4, Push 25
20.0.0.0/24        *[Direct/0] 04:59:10
                    >  via jvknet1-808ed81
20.0.0.6/32        *[Local/0] 04:59:10
                       Local via jvknet1-808ed81

root@jcnr2> show route table mpls.0 label 18 detail

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
18  (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x55aac602687c
                Next-hop reference count: 3, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-808ed81, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55aac8e1b558
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 4:08:01
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (3): 1-KRT MFS 2-KRT 4-KRT-vRouter
                AS path: I
		Ref Cnt: 1
                Thread: junos-main

18(S=0) (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x55aac602679c
                Next-hop reference count: 1, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-808ed81, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55aac8e1c488
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 4:08:01
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (1): 1-KRT MFS
                AS path: I
		Ref Cnt: 1
                Thread: junos-main
```


vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-dpjmx -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:116 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:116  bytes:10272 errors:0
            TX packets:128  bytes:11728 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:14
            RX device packets:31348  bytes:3005971 errors:0
            RX port   packets:31348 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:31348  bytes:3005971 errors:0
            TX packets:12042  bytes:1039277 errors:0
            Drops:0
            TX queue  packets:11997 errors:0
            TX port   packets:12042 errors:0
            TX device packets:12063  bytes:1041192 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:7  bytes:826 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:7  bytes:826 errors:7
            TX packets:0  bytes:0 errors:0
            Drops:7

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2 QOS:0 Ref:13 TxXVif:1
            RX device packets:11520  bytes:995129 errors:0
            RX queue  packets:11520 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:11520  bytes:995129 errors:0
            TX packets:18865  bytes:1735249 errors:0
            Drops:0
            TX queue  packets:18865 errors:0
            TX device packets:18865  bytes:1735249 errors:0

vif0/6      Ethernet: jvknet1-37a41a5 NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:10.0.0.37
            IP6addr:abcd::a00:2
            DDP: OFF SwLB: ON
            Vrf:2 Mcast Vrf:2 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:275 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:275  bytes:26674 errors:0
            TX packets:212  bytes:20404 errors:0
            Drops:57
            TX queue  packets:203 errors:0
            TX port   packets:212 errors:0
```
```
[root@jcnr2 ~]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:253 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:253  bytes:22022 errors:0
            TX packets:440  bytes:38664 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:47227  bytes:4024471 errors:0
            RX port   packets:47215 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:47215  bytes:4022051 errors:0
            TX packets:79363  bytes:7353072 errors:0
            Drops:0
            TX queue  packets:79363 errors:0
            TX port   packets:79363 errors:0
            TX device packets:79385  bytes:7355044 errors:0

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
            RX device packets:78941  bytes:7324944 errors:0
            RX queue  packets:78941 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:78941  bytes:7324944 errors:0
            TX packets:47046  bytes:4004813 errors:0
            Drops:0
            TX queue  packets:47046 errors:0
            TX device packets:47046  bytes:4004813 errors:0

vif0/6      Ethernet: jvknet1-808ed81 NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:20.0.0.6
            IP6addr:abcd::1400:6
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:300 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:300  bytes:43984 errors:0
            TX packets:177  bytes:16986 errors:0
            Drops:117
            TX queue  packets:169 errors:0
            TX port   packets:177 errors:0
```

POD間疎通確認
```
[root@vrf1-pod1 /]# ping 20.0.0.6
PING 20.0.0.6 (20.0.0.6) 56(84) bytes of data.
64 bytes from 20.0.0.6: icmp_seq=1 ttl=62 time=1.30 ms
64 bytes from 20.0.0.6: icmp_seq=2 ttl=62 time=0.265 ms
64 bytes from 20.0.0.6: icmp_seq=3 ttl=62 time=0.245 ms
```
