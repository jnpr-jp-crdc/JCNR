# VRF - VRF Lite : JCNR間Routing
- VRF Instance Type : Virtual Router
- JCNR WorkerNode間でPOD間L3接続
- POD / JCNR vRouter間はL3接続

## VRF Lite - Pod Vlan Access
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vrouter1.png" width=800>

### Network Attachment Definition　作成
[VRF Lite NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-nad-jcnr1.yaml)

[VRF Lite NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-nad-jcnr2.yaml)

#### NAD Option
- InstanceType : VRF Liteは"virtual-router"を使用
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
- NADで指定したRouteがStatic Routeとして反映される
```
[root@jcnr1]# kubectl describe pod vrouter1-pod1
Name:         vrouter1-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Thu, 05 Oct 2023 02:08:49 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: f0d04b6a2973afe6d977e0c2bb5003ae4f650556fe88e36267af3b75cb78fbfa
              cni.projectcalico.org/podIP: 172.30.79.42/32
              cni.projectcalico.org/podIPs: 172.30.79.42/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.42"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrouter1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.16",
                        "abcd::a00:b"
                    ],
                    "mac": "02:00:00:F8:8C:50",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrouter1",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```
```
[root@jcnr2]# kubectl describe pod vrouter1-pod2
Name:         vrouter1-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Thu, 05 Oct 2023 02:28:46 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 9932edeb4629c8e23bf2c6844b45bef05303ff3b6da890b44721c14608ba03c3
              cni.projectcalico.org/podIP: 172.30.147.244/32
              cni.projectcalico.org/podIPs: 172.30.147.244/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.244"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrouter1",
                    "interface": "net1",
                    "ips": [
                        "20.0.0.3",
                        "abcd::1400:3"
                    ],
                    "mac": "02:00:00:CB:8C:63",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrouter1",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```

POD IP/Route 確認
```
[root@vrouter1-pod1 /]# ip a
--- snip ---
269: net1@if270: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:f8:8c:50 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.16/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:b/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fef8:8c50/64 scope link
       valid_lft forever preferred_lft forever

[root@vrouter1-pod1 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 10.0.0.1 dev net1
10.0.0.1 dev net1 scope link
20.0.0.0/24 via 10.0.0.1 dev net1
169.254.1.1 dev eth0 scope link

[root@vrouter1-pod1 /]# ip -6 route
abcd::a00:1 dev net1 metric 1024 pref medium
abcd::a00:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
abcd::1400:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev net1 proto kernel metric 256 pref medium
```
```
[root@vrouter1-pod2 /]# ip a
--- snip ---
120: net1@if121: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:cb:8c:63 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.3/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fecb:8c63/64 scope link
       valid_lft forever preferred_lft forever

[root@vrouter1-pod2 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.1 dev net1 scope link
169.254.1.1 dev eth0 scope link

[root@vrouter1-pod2 /]# ip -6 route
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
set groups cni routing-instances vrouter1 instance-type virtual-router
set groups cni routing-instances vrouter1 routing-options rib vrouter1.inet6.0 static route abcd::a00:b/128 qualified-next-hop abcd::a00:b interface jvknet1-bdc2b2b
set groups cni routing-instances vrouter1 routing-options static route 10.0.0.16/32 qualified-next-hop 10.0.0.16 interface jvknet1-bdc2b2b
set groups cni routing-instances vrouter1 interface jvknet1-bdc2b2b
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vrouter1 instance-type virtual-router
set groups cni routing-instances vrouter1 routing-options rib vrouter1.inet6.0 static route abcd::1400:3/128 qualified-next-hop abcd::1400:3 interface jvknet1-0cfa61d
set groups cni routing-instances vrouter1 routing-options static route 20.0.0.3/32 qualified-next-hop 20.0.0.3 interface jvknet1-0cfa61d
set groups cni routing-instances vrouter1 interface jvknet1-0cfa61d
```

cRPD Interface ens4設定追加
- Pod/JCNR vRouter間でSub Interfaceを使用していないため、Physical InterfaceではSub Interface使用不可
```
root@jcnr1> show configuration | display set
set interfaces ens4 unit 0 family inet address 192.168.0.1/24
set routing-instances vrouter1 routing-options static route 20.0.0.0/24 next-hop 192.168.0.2
set routing-instances vrouter1 interface ens4
```
```
root@jcnr2> show configuration | display set
set interfaces ens4 unit 0 family inet address 192.168.0.2/24
set routing-instances vrouter1 routing-options static route 10.0.0.0/24 next-hop 192.168.0.1
set routing-instances vrouter1 interface ens4
```

cRPD Route確認
```
root@jcnr1> show route table vrouter1.inet.0

vrouter1.inet.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 01:03:26
                    >  via jvknet1-bdc2b2b
10.0.0.16/32       *[Local/0] 01:03:26
                       Local via jvknet1-bdc2b2b
20.0.0.0/24        *[Static/5] 00:22:56
                    >  to 192.168.0.2 via ens4
192.168.0.0/24     *[Direct/0] 00:22:56
                    >  via ens4
192.168.0.1/32     *[Local/0] 00:22:56
                       Local via ens4
```
```
root@jcnr2> show route table vrouter1.inet.0

vrouter1.inet.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Static/5] 00:26:38
                    >  to 192.168.0.1 via ens4
20.0.0.0/24        *[Direct/0] 00:44:05
                    >  via jvknet1-0cfa61d
20.0.0.3/32        *[Local/0] 00:44:05
                       Local via jvknet1-0cfa61d
192.168.0.0/24     *[Direct/0] 00:26:38
                    >  via ens4
192.168.0.2/32     *[Local/0] 00:26:38
                       Local via ens4
```


vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-dpjmx -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:326 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:326  bytes:28096 errors:0
            TX packets:59315  bytes:8404890 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:L3L2Vof QOS:0 Ref:14
            RX device packets:84562  bytes:6762076 errors:0
            RX port   packets:84560 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:84560  bytes:6761908 errors:0
            TX packets:56082  bytes:4656752 errors:0
            Drops:17
            TX queue  packets:55902 errors:0
            TX port   packets:56082 errors:0
            TX device packets:56093  bytes:4657888 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:171  bytes:13406 errors:0
            RX port   packets:171 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:171  bytes:13790 errors:0
            TX packets:198  bytes:14760 errors:0
            Drops:61
            TX queue  packets:198 errors:0
            TX port   packets:198 errors:0
            TX device packets:198  bytes:14760 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:15 TxXVif:1
            RX device packets:54894  bytes:4566975 errors:0
            RX queue  packets:54894 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:54894  bytes:4566975 errors:0
            TX packets:84146  bytes:6731207 errors:0
            Drops:0
            TX queue  packets:84146 errors:0
            TX device packets:84146  bytes:6731207 errors:0

vif0/6      Ethernet: jvknet1-bdc2b2b NH: 16 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:10.0.0.16
            IP6addr:abcd::a00:b
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:653 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:653  bytes:61198 errors:0
            TX packets:224  bytes:20876 errors:0
            Drops:135
            TX queue  packets:204 errors:0
            TX port   packets:224 errors:0
```
```
[root@jcnr2 ~]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:1759 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:1759  bytes:152770 errors:0
            TX packets:2003  bytes:190962 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:2 Mcast Vrf:2 Flags:L3L2Vof QOS:0 Ref:16
            RX device packets:115154  bytes:9550355 errors:0
            RX port   packets:115142 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:115142  bytes:9546568 errors:0
            TX packets:179897  bytes:15482418 errors:0
            Drops:140
            TX queue  packets:178875 errors:0
            TX port   packets:179897 errors:0
            TX device packets:179924  bytes:15484748 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:ac:04:91
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:244  bytes:18728 errors:0
            RX port   packets:244 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:217  bytes:17850 errors:27
            TX packets:265  bytes:27681 errors:0
            Drops:65
            TX queue  packets:265 errors:0
            TX port   packets:265 errors:0
            TX device packets:265  bytes:27681 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:192.168.0.2
            DDP: OFF SwLB: ON
            Vrf:2 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:17 TxXVif:1
            RX device packets:169931  bytes:14627746 errors:0
            RX queue  packets:169931 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:169931  bytes:14627746 errors:0
            TX packets:113070  bytes:9395160 errors:0
            Drops:0
            TX queue  packets:113070 errors:0
            TX device packets:113070  bytes:9395160 errors:0

vif0/6      Ethernet: jvknet1-0cfa61d NH: 14 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:20.0.0.3
            IP6addr:abcd::1400:3
            DDP: OFF SwLB: ON
            Vrf:2 Mcast Vrf:2 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:565 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:565  bytes:68763 errors:0
            TX packets:465  bytes:44774 errors:0
            Drops:332
            TX queue  packets:450 errors:0
            TX port   packets:465 errors:0
```

POD間疎通確認
```
[root@vrouter1-pod1 /]# ping 20.0.0.3
PING 20.0.0.3 (20.0.0.3) 56(84) bytes of data.
64 bytes from 20.0.0.3: icmp_seq=1 ttl=62 time=1.61 ms
64 bytes from 20.0.0.3: icmp_seq=2 ttl=62 time=0.268 ms
64 bytes from 20.0.0.3: icmp_seq=3 ttl=62 time=0.276 ms
```
