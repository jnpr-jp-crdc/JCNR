# VRF - VRF L3VPN : JCNR間Routing
- VRF Instance Type : vrf
- JCNR WorkerNode間でPOD間MPLSoUDPによるL3接続
- POD / JCNR vRouter間はL3接続

## VRF L3VPN - JCNR間Routing
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vrf1.png" width=800>

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
Start Time:   Fri, 06 Oct 2023 00:56:07 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: c5dbb01f484acae5e26e1732f82bfd8fb6c8bb506d1dbb256b1b276fb9cc85c1
              cni.projectcalico.org/podIP: 172.30.79.57/32
              cni.projectcalico.org/podIPs: 172.30.79.57/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.57"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.37",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:48:75:1F",
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
Start Time:   Fri, 06 Oct 2023 00:56:23 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: d61ba9535f253ec3e93393f040905bb52c3c646194bb844c501e8cdb0c0f4381
              cni.projectcalico.org/podIP: 172.30.147.250/32
              cni.projectcalico.org/podIPs: 172.30.147.250/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.250"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrf1",
                    "interface": "net1",
                    "ips": [
                        "20.0.0.2",
                        "abcd::1400:2"
                    ],
                    "mac": "02:00:00:7D:FD:33",
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
320: net1@if321: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:48:75:1f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.37/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe48:751f/64 scope link
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
160: net1@if161: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:7d:fd:33 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.2/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe7d:fd33/64 scope link
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
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::a00:2/128 qualified-next-hop abcd::a00:2 interface jvknet1-37a41a5
set groups cni routing-instances vrf1 routing-options static route 10.0.0.37/32 qualified-next-hop 10.0.0.37 interface jvknet1-37a41a5
set groups cni routing-instances vrf1 interface jvknet1-37a41a5
set groups cni routing-instances vrf1 vrf-target target:64512:1
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vrf1 instance-type vrf
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::1400:2/128 qualified-next-hop abcd::1400:2 interface jvknet1-09f49c1
set groups cni routing-instances vrf1 routing-options static route 20.0.0.2/32 qualified-next-hop 20.0.0.2 interface jvknet1-09f49c1
set groups cni routing-instances vrf1 interface jvknet1-09f49c1
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
1.1.1.2               64512        686        690       0       0     5:04:44 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/1/1/0
  vrf1.inet.0: 1/1/1/0
  vrf1.inet6.0: 0/1/1/0

root@jcnr1> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 01:10:47
                    >  via jvknet1-37a41a5
10.0.0.37/32       *[Local/0] 01:10:47
                       Local via jvknet1-37a41a5
20.0.0.0/24        *[BGP/170] 01:10:29, localpref 100, from 1.1.1.2
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.2 via ens4, Push 31
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
1.1.1.1               64512        693        686       0       0     5:05:29 Establ
  bgp.l3vpn.0: 1/1/1/0
  bgp.l3vpn-inet6.0: 0/1/1/0
  vrf1.inet.0: 1/1/1/0
  vrf1.inet6.0: 0/1/1/0

root@jcnr2> show route table vrf1.inet.0

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[BGP/170] 00:00:56, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 192.168.0.1 via ens4, Push 36
20.0.0.0/24        *[Direct/0] 00:00:57
                    >  via jvknet1-09f49c1
20.0.0.2/32        *[Local/0] 00:00:57
                       Local via jvknet1-09f49c1
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
            RX port   packets:2 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:2  bytes:216 errors:0
            TX packets:4  bytes:424 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:12
            RX device packets:158  bytes:18858 errors:0
            RX port   packets:149 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:149  bytes:16677 errors:0
            TX packets:221  bytes:22660 errors:0
            Drops:0
            TX queue  packets:221 errors:0
            TX port   packets:221 errors:0
            TX device packets:243  bytes:24681 errors:0

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
            RX device packets:219  bytes:22532 errors:0
            RX queue  packets:219 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:219  bytes:22532 errors:0
            TX packets:149  bytes:16677 errors:0
            Drops:0
            TX queue  packets:149 errors:0
            TX device packets:149  bytes:16677 errors:0

vif0/6      Ethernet: jvknet1-09f49c1 NH: 20 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:20.0.0.2
            IP6addr:abcd::1400:2
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:16 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:16  bytes:1416 errors:0
            TX packets:3  bytes:258 errors:0
            Drops:13
            TX port   packets:3 errors:0
```

POD間疎通確認
```
[root@vrf1-pod1 /]# ping 20.0.0.2
PING 20.0.0.2 (20.0.0.2) 56(84) bytes of data.
64 bytes from 20.0.0.2: icmp_seq=1 ttl=62 time=1.30 ms
64 bytes from 20.0.0.2: icmp_seq=2 ttl=62 time=0.265 ms
64 bytes from 20.0.0.2: icmp_seq=3 ttl=62 time=0.245 ms
```
