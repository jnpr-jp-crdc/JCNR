# VRF - VRF L3VPN / MPLSoUDP : JCNR間Routing
- VRF Instance Type : vrf
- JCNR WorkerNode間でPOD間MPLSoUDPによるL3接続
- POD / JCNR vRouter間はL3接続

## VRF L3VPN / MPLSoUDP - JCNR間Routing
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vrf1.png" width=800>

## JCNR Configlet

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
Start Time:   Tue, 17 Oct 2023 02:36:46 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 77735fbd58f3ad820e070f15d248c4a40cc2db6513bb791fa4bd4170efae0486
              cni.projectcalico.org/podIP: 172.30.79.20/32
              cni.projectcalico.org/podIPs: 172.30.79.20/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.20"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.55",
                        "abcd::a00:14"
                    ],
                    "mac": "02:00:00:00:48:CD",
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
Start Time:   Tue, 17 Oct 2023 02:39:58 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: ed2f35055fe53e087b856a50656cc2b5fc87c11e3606312c333589864ad30718
              cni.projectcalico.org/podIP: 172.30.147.255/32
              cni.projectcalico.org/podIPs: 172.30.147.255/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.255"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "20.0.0.7",
                        "abcd::1400:7"
                    ],
                    "mac": "02:00:00:6E:B4:D5",
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
435: net1@if436: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:00:48:cd brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.55/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:14/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe00:48cd/64 scope link
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
201: net1@if202: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:6e:b4:d5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.7/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:7/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe6e:b4d5/64 scope link
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
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::a00:14/128 qualified-next-hop abcd::a00:14 interface jvknet1-509adaa
set groups cni routing-instances vrf1 routing-options static route 10.0.0.55/32 qualified-next-hop 10.0.0.55 interface jvknet1-509adaa
set groups cni routing-instances vrf1 interface jvknet1-509adaa
set groups cni routing-instances vrf1 vrf-target target:64512:1
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vrf1 instance-type vrf
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::1400:7/128 qualified-next-hop abcd::1400:7 interface jvknet1-b553603
set groups cni routing-instances vrf1 routing-options static route 20.0.0.7/32 qualified-next-hop 20.0.0.7 interface jvknet1-b553603
set groups cni routing-instances vrf1 interface jvknet1-b553603
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
bgp.l3vpn.0
                       1          1          0          0          0          0
bgp.l3vpn-inet6.0
                       1          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
1.1.1.2               64512         40         41       0       0       14:49 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/1/1/0
  vrf1.inet.0: 1/1/1/0
  vrf1.inet6.0: 0/1/1/0

root@jcnr1> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 00:13:02
                    >  via jvknet1-509adaa
10.0.0.55/32       *[Local/0] 00:13:02
                       Local via jvknet1-509adaa
20.0.0.0/24        *[BGP/170] 00:08:30, localpref 100, from 1.1.1.2
                      AS path: I, validation-state: unverified
                    >  via Tunnel Composite, UDP (src 1.1.1.1 dest 1.1.1.2), Push 21

root@jcnr1> show route table mpls.0

mpls.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

18                 *[VPN/170] 00:14:17
                    >  via jvknet1-509adaa, Pop
18(S=0)            *[VPN/170] 00:14:17
                    >  via jvknet1-509adaa, Pop
19                 *[VPN/170] 00:14:17
                    >  via jvknet1-509adaa, Pop
19(S=0)            *[VPN/170] 00:14:17
                    >  via jvknet1-509adaa, Pop

root@jcnr1> show route table mpls.0 label 19 detail

mpls.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
19  (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x55886bc2bffc
                Next-hop reference count: 3, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-509adaa, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55886ea1b510
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 14:45
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (3): 1-KRT MFS 2-KRT 4-KRT-vRouter
                AS path: I
		Ref Cnt: 1
                Thread: junos-main

19(S=0) (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x55886bc2c0dc
                Next-hop reference count: 1, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-509adaa, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55886ea1b4c8
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 14:45
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (1): 1-KRT MFS
                AS path: I
		Ref Cnt: 1
                Thread: junos-main

```
```
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
1.1.1.1               64512         43         39       0       2       15:25 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/1/1/0
  vrf1.inet.0: 1/1/1/0
  vrf1.inet6.0: 0/1/1/0

root@jcnr2> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[BGP/170] 00:09:05, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  via Tunnel Composite, UDP (src 1.1.1.2 dest 1.1.1.1), Push 19
20.0.0.0/24        *[Direct/0] 00:10:22
                    >  via jvknet1-b553603
20.0.0.7/32        *[Local/0] 00:10:22
                       Local via jvknet1-b553603

root@jcnr2> show route table mpls.0

mpls.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

20                 *[VPN/170] 00:10:42
                    >  via jvknet1-b553603, Pop
20(S=0)            *[VPN/170] 00:10:42
                    >  via jvknet1-b553603, Pop
21                 *[VPN/170] 00:10:42
                    >  via jvknet1-b553603, Pop
21(S=0)            *[VPN/170] 00:10:42
                    >  via jvknet1-b553603, Pop

root@jcnr2> show route table mpls.0 label 21 detail

mpls.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
21  (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x55aac602609c
                Next-hop reference count: 3, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-b553603, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55aac8e1bd38
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 11:54
                Validation State: unverified
                Task: BGP_RT_Background
                Announcement bits (3): 1-KRT MFS 2-KRT 4-KRT-vRouter
                AS path: I
		Ref Cnt: 1
                Thread: junos-main

21(S=0) (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 0
                Address: 0x55aac60265dc
                Next-hop reference count: 1, Next-hop session id: 0
                Kernel Table Id: 0
                Next hop: via jvknet1-b553603, selected
                Label operation: Pop
                Load balance label: None;
                Label element ptr: 0x55aac8e1bd80
                Label parent element ptr: (nil)
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0
                State: <Active Int Ext>
                Age: 11:54
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
            RX port   packets:4 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:4  bytes:388 errors:0
            TX packets:17  bytes:1698 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:748  bytes:79044 errors:0
            RX port   packets:744 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:744  bytes:77216 errors:0
            TX packets:449  bytes:43576 errors:0
            Drops:0
            TX queue  packets:449 errors:0
            TX port   packets:449 errors:0
            TX device packets:469  bytes:45423 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
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
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:1
            RX device packets:398  bytes:37254 errors:0
            RX queue  packets:398 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:398  bytes:37254 errors:0
            TX packets:697  bytes:71106 errors:0
            Drops:0
            TX queue  packets:697 errors:0
            TX device packets:697  bytes:71106 errors:0

vif0/6      Ethernet: jvknet1-509adaa NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:10.0.0.55
            IP6addr:abcd::a00:14
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:78 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:78  bytes:6916 errors:0
            TX packets:50  bytes:4776 errors:0
            Drops:26
            TX queue  packets:47 errors:0
            TX port   packets:50 errors:0
```
```
[root@jcnr2 ~]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:257 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:257  bytes:22410 errors:0
            TX packets:481  bytes:42554 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:47910  bytes:4090147 errors:0
            RX port   packets:47898 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:47898  bytes:4087727 errors:0
            TX packets:80640  bytes:7485156 errors:0
            Drops:0
            TX queue  packets:80640 errors:0
            TX port   packets:80640 errors:0
            TX device packets:80662  bytes:7487128 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:ac:04:91
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:8  bytes:936 errors:0
            RX port   packets:8 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:8  bytes:968 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:1

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:11 TxXVif:1
            RX device packets:80167  bytes:7450706 errors:0
            RX queue  packets:80167 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:80167  bytes:7450706 errors:0
            TX packets:47682  bytes:4064379 errors:0
            Drops:0
            TX queue  packets:47682 errors:0
            TX device packets:47682  bytes:4064379 errors:0

vif0/6      Ethernet: jvknet1-b553603 NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:20.0.0.7
            IP6addr:abcd::1400:7
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:154 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:154  bytes:25058 errors:0
            TX packets:51  bytes:4818 errors:0
            Drops:100
            TX queue  packets:47 errors:0
            TX port   packets:51 errors:0
```

POD間疎通確認
```
[root@vrf1-pod1 /]# ping 20.0.0.7
PING 20.0.0.7 (20.0.0.7) 56(84) bytes of data.
64 bytes from 20.0.0.7: icmp_seq=1 ttl=62 time=1.12 ms
64 bytes from 20.0.0.7: icmp_seq=2 ttl=62 time=0.294 ms
64 bytes from 20.0.0.7: icmp_seq=3 ttl=62 time=0.287 ms
```
