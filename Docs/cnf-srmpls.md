# CNF Mode - SR-MPLS
- JCNRはCNF Modeとして稼働し、CE間のL3接続
- JCNR間でSR-MPLSによるL3接続

## CNF Mode - SR-MPLS
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/cnf-sr-mpls.png" width=800>

cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set interfaces ens5 unit 0 family inet address 192.168.1.1/24
set routing-instances vrf1 instance-type vrf
set routing-instances vrf1 interface ens5
set routing-instances vrf1 vrf-target target:64512:1
set routing-instances vrf1 vrf-table-label
```
```
root@jcnr2> show configuration | display set
--- snip ---
set interfaces ens5 unit 0 family inet address 192.168.2.1/24
set routing-instances vrf1 instance-type vrf
set routing-instances vrf1 interface ens5
set routing-instances vrf1 vrf-target target:64512:1
set routing-instances vrf1 vrf-table-label
```

cRPD Route確認
```
root@jcnr1> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ens4                  jcnr2          2  Up                     6  52:54:0:cf:74:89

root@jcnr1> show isis database detail
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

jcnr1.00-00 Sequence: 0x94, Checksum: 0x25a7, Lifetime: 1037 secs
  IPV4 Index: 2001
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      20, Weight:   0, Neighbor: jcnr2, Flags: --VL--
     LAN IPv6 Adj-SID:      21, Weight:   0, Neighbor: jcnr2, Flags: F-VL--
   IP prefix: 1.1.1.1/32                      Metric:        0 Internal Up
   IP prefix: 169.254.119.159/32              Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::60e5:a2ff:fe19:dd4/128    Metric:        0 Internal Up

jcnr2.00-00 Sequence: 0x9b, Checksum: 0x9178, Lifetime: 592 secs
  IPV4 Index: 2002
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      16, Weight:   0, Neighbor: jcnr1, Flags: --VL--
     LAN IPv6 Adj-SID:      17, Weight:   0, Neighbor: jcnr1, Flags: F-VL--
   IP prefix: 1.1.1.2/32                      Metric:        0 Internal Up
   IP prefix: 169.254.27.155/32               Metric:       10 Internal Up
   IP prefix: 172.30.147.192/32               Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::d882:4fff:fef9:a3c3/128   Metric:        0 Internal Up

jcnr2.02-00 Sequence: 0x87, Checksum: 0x7f53, Lifetime: 475 secs
   IS neighbor: jcnr1.00                      Metric:        0
   IS neighbor: jcnr2.00                      Metric:        0

root@jcnr1> show route table mpls.0

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 07:17:09, metric 1
                       Receive
1                  *[MPLS/0] 07:17:09, metric 1
                       Receive
2                  *[MPLS/0] 07:17:09, metric 1
                       Receive
13                 *[MPLS/0] 07:17:09, metric 1
                       Receive
20                 *[L-ISIS/14] 06:54:12, metric 0
                    >  to 192.168.0.2 via ens4, Pop
20(S=0)            *[L-ISIS/14] 06:54:12, metric 0
                    >  to 192.168.0.2 via ens4, Pop
21                 *[L-ISIS/14] 06:54:12, metric 0
                    >  to fe80::38a0:6eff:fecc:5f77 via ens4, Pop
21(S=0)            *[L-ISIS/14] 06:54:12, metric 0
                    >  to fe80::38a0:6eff:fecc:5f77 via ens4, Pop
28                 *[VPN/0] 00:15:52
                    >  via __crpd-vrf2 (vrf1), Pop
402002             *[L-ISIS/14] 06:54:11, metric 10
                    >  to 192.168.0.2 via ens4, Pop
402002(S=0)        *[L-ISIS/14] 06:54:11, metric 10
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
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.2               64512        951        945       0       1     6:54:00 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/0/0/0
  vrf1.inet.0: 1/1/1/0

root@jcnr1> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.0/24     *[Direct/0] 00:24:15
                    >  via ens5
192.168.1.1/32     *[Local/0] 00:24:15
                       Local via ens5
192.168.2.0/24     *[BGP/170] 00:15:35, localpref 100, from 1.1.1.2
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.2 via ens4, Push 24

root@jcnr1> show route table mpls.0 label 28 detail

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
28  (1 entry, 1 announced)
        *VPN    Preference: 0
                Next hop type: Router, Next hop index: 0
                Address: 0x55f3b55b42bc
                Next-hop reference count: 3, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via __crpd-vrf2 (vrf1), selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55f3b061c098
                Label parent element ptr: (nil)
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext LsiL3>
                Age: 17:00
                Validation State: unverified
                Task: RT
                Announcement bits (3): 1-KRT MFS 2-KRT 4-KRT-vRouter
                AS path: I
		Ref Cnt: 131924078
                Thread: junos-main
```
```
root@jcnr2> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ens4                  jcnr1          2  Up                    23  52:54:0:43:f9:a1

root@jcnr2> show isis database detail
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

jcnr1.00-00 Sequence: 0x94, Checksum: 0x25a7, Lifetime: 909 secs
  IPV4 Index: 2001
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      20, Weight:   0, Neighbor: jcnr2, Flags: --VL--
     LAN IPv6 Adj-SID:      21, Weight:   0, Neighbor: jcnr2, Flags: F-VL--
   IP prefix: 1.1.1.1/32                      Metric:        0 Internal Up
   IP prefix: 169.254.119.159/32              Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::60e5:a2ff:fe19:dd4/128    Metric:        0 Internal Up

jcnr2.00-00 Sequence: 0x9b, Checksum: 0x9178, Lifetime: 468 secs
  IPV4 Index: 2002
  Node Segment Blocks Advertised:
    Start Index : 0, Size : 4000, Label-Range: [ 400000, 403999 ]
   IS neighbor: jcnr2.02                      Metric:       10
     LAN IPv4 Adj-SID:      16, Weight:   0, Neighbor: jcnr1, Flags: --VL--
     LAN IPv6 Adj-SID:      17, Weight:   0, Neighbor: jcnr1, Flags: F-VL--
   IP prefix: 1.1.1.2/32                      Metric:        0 Internal Up
   IP prefix: 169.254.27.155/32               Metric:       10 Internal Up
   IP prefix: 172.30.147.192/32               Metric:       10 Internal Up
   IP prefix: 192.168.0.0/24                  Metric:       10 Internal Up
   V6 prefix: ::/96                           Metric:       10 Internal Up
   V6 prefix: fe80::d882:4fff:fef9:a3c3/128   Metric:        0 Internal Up

jcnr2.02-00 Sequence: 0x88, Checksum: 0x7d54, Lifetime: 1110 secs
   IS neighbor: jcnr1.00                      Metric:        0
   IS neighbor: jcnr2.00                      Metric:        0

root@jcnr2> show route table mpls.0

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 06:56:33, metric 1
                       Receive
1                  *[MPLS/0] 06:56:33, metric 1
                       Receive
2                  *[MPLS/0] 06:56:33, metric 1
                       Receive
13                 *[MPLS/0] 06:56:33, metric 1
                       Receive
16                 *[L-ISIS/14] 06:56:16, metric 0
                    >  to 192.168.0.1 via ens4, Pop
16(S=0)            *[L-ISIS/14] 06:56:16, metric 0
                    >  to 192.168.0.1 via ens4, Pop
17                 *[L-ISIS/14] 06:56:16, metric 0
                    >  to fe80::5054:ff:fe43:f9a1 via ens4, Pop
17(S=0)            *[L-ISIS/14] 06:56:16, metric 0
                    >  to fe80::5054:ff:fe43:f9a1 via ens4, Pop
24                 *[VPN/0] 00:17:08
                    >  via __crpd-vrf2 (vrf1), Pop
402001             *[L-ISIS/14] 06:56:16, metric 10
                    >  to 192.168.0.1 via ens4, Pop
402001(S=0)        *[L-ISIS/14] 06:56:16, metric 10
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
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.1               64512        950        953       0       0     6:56:04 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/0/0/0
  vrf1.inet.0: 1/1/1/0

root@jcnr2> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.0/24     *[BGP/170] 00:18:24, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.1 via ens4, Push 28
192.168.2.0/24     *[Direct/0] 00:25:29
                    >  via ens5
192.168.2.1/32     *[Local/0] 00:25:29
                       Local via ens5

root@jcnr2> show route table mpls.0 label 24 detail

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
24  (1 entry, 1 announced)
        *VPN    Preference: 0
                Next hop type: Router, Next hop index: 0
                Address: 0x562b2f82721c
                Next-hop reference count: 3, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via __crpd-vrf2 (vrf1), selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x562b3261b1b0
                Label parent element ptr: (nil)
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext LsiL3>
                Age: 18:00
                Validation State: unverified
                Task: RT
                Announcement bits (3): 1-KRT MFS 2-KRT 4-KRT-vRouter
                AS path: I
		Ref Cnt: 131924078
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
            RX port   packets:95 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:95  bytes:8478 errors:0
            TX packets:137  bytes:12382 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:25452  bytes:2350963 errors:0
            RX port   packets:25447 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:25447  bytes:2350543 errors:0
            TX packets:16456  bytes:1399931 errors:0
            Drops:0
            TX queue  packets:16456 errors:0
            TX port   packets:16456 errors:0
            TX device packets:16467  bytes:1400993 errors:0

vif0/2      PCI: 0000:00:05.0 NH: 7 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8 IPaddr:192.168.1.1
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:L3L2Vof QOS:0 Ref:10
            RX device packets:1927  bytes:187998 errors:0
            RX port   packets:1927 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            RX packets:1927  bytes:187998 errors:0
            TX packets:9294  bytes:811610 errors:0
            Drops:1427
            TX queue  packets:9294 errors:0
            TX port   packets:9294 errors:0
            TX device packets:9298  bytes:812042 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:1
            RX device packets:15888  bytes:1348823 errors:0
            RX queue  packets:15888 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:15888  bytes:1348823 errors:0
            TX packets:24991  bytes:2305795 errors:0
            Drops:0
            TX queue  packets:24991 errors:0
            TX device packets:24991  bytes:2305795 errors:0

vif0/4      PMD: ens5 NH: 15 MTU: 9000
            Type:Host HWaddr:52:54:00:60:b1:d8 IPaddr:192.168.1.1
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:2
            RX device packets:8831  bytes:766628 errors:0
            RX queue  packets:8831 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:8831  bytes:766628 errors:0
            TX packets:20  bytes:1112 errors:0
            Drops:0
            TX queue  packets:20 errors:0
            TX device packets:20  bytes:1112 errors:0
```
```
[root@jcnr2 ~]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:89 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:89  bytes:7742 errors:0
            TX packets:140  bytes:13912 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:15788  bytes:1339980 errors:0
            RX port   packets:15781 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:15781  bytes:1339377 errors:0
            TX packets:25066  bytes:2311716 errors:0
            Drops:22
            TX queue  packets:25066 errors:0
            TX port   packets:25066 errors:0
            TX device packets:25081  bytes:2313156 errors:0

vif0/2      PCI: 0000:00:05.0 NH: 7 MTU: 9000
            Type:Physical HWaddr:52:54:00:ac:04:91 IPaddr:192.168.2.1
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:L3L2Vof QOS:0 Ref:10
            RX device packets:487  bytes:46270 errors:0
            RX port   packets:487 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            RX packets:487  bytes:46270 errors:0
            TX packets:8834  bytes:769620 errors:0
            Drops:0
            TX queue  packets:8834 errors:0
            TX port   packets:8834 errors:0
            TX device packets:8842  bytes:770532 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:1
            RX device packets:24527  bytes:2263222 errors:0
            RX queue  packets:24527 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:24527  bytes:2263222 errors:0
            TX packets:15303  bytes:1292473 errors:0
            Drops:0
            TX queue  packets:15303 errors:0
            TX device packets:15303  bytes:1292473 errors:0

vif0/4      PMD: ens5 NH: 15 MTU: 9000
            Type:Host HWaddr:52:54:00:ac:04:91 IPaddr:192.168.2.1
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:2
            RX device packets:8369  bytes:724554 errors:0
            RX queue  packets:8369 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:8369  bytes:724554 errors:0
            TX packets:28  bytes:1288 errors:0
            Drops:0
            TX queue  packets:28 errors:0
            TX device packets:28  bytes:1288 errors:0
```

CE間疎通確認
```
root@ce1> ping 192.168.2.2
PING 192.168.2.2 (192.168.2.2): 56 data bytes
64 bytes from 192.168.2.2: icmp_seq=0 ttl=62 time=3.554 ms
64 bytes from 192.168.2.2: icmp_seq=1 ttl=62 time=1.685 ms
```
