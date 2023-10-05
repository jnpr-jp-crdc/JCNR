# VRF - VRF Lite : virtual-router間接続
- VRF Instance Type : Virtual Router
- JCNR WorkerNode内でPOD間L3接続
- Virtual Router間のRoute Leak
- POD / JCNR vRouter間はL3接続

## VRF Lite - virtual-router間接続
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vrouter2.png" width=800>

### Network Attachment Definition　作成
[VRF Lite vrouter2-1 NAD Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter2-1-nad.yaml)

[VRF Lite vrouter2-2 NAD Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter2-2-nad.yaml)

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
[JCNR1 vrouter2-1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter2-1-pod1.yaml)

[JCNR2 vrouter2-2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter2-2-pod2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
- NADで指定したRouteがStatic Routeとして反映される
```
[root@jcnr1 yaml]# kubectl describe pod vrouter2-1-pod1
Name:         vrouter2-1-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Thu, 05 Oct 2023 03:33:00 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 1f4d53592901b7250aa7972b5fb4a901048fb6972fd274a0b9eba9d182bf80e4
              cni.projectcalico.org/podIP: 172.30.79.46/32
              cni.projectcalico.org/podIPs: 172.30.79.46/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.46"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrouter2-1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:2E:2F:1F",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrouter2-1",
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
[root@jcnr1 yaml]# kubectl describe pod vrouter2-2-pod2
Name:         vrouter2-2-pod2
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Thu, 05 Oct 2023 03:40:12 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: cb2512fbb48f769822de8cbba6a7e2bf335ebfb0313270ae7e32935480e52a65
              cni.projectcalico.org/podIP: 172.30.79.48/32
              cni.projectcalico.org/podIPs: 172.30.79.48/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.48"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrouter2-2",
                    "interface": "net1",
                    "ips": [
                        "20.0.0.3",
                        "abcd::1400:3"
                    ],
                    "mac": "02:00:00:17:D2:43",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrouter2-2",
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
[root@vrouter2-1-pod1 /]# ip a
--- snip ---
275: net1@if276: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:2e:2f:1f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe2e:2f1f/64 scope link
       valid_lft forever preferred_lft forever

[root@vrouter2-1-pod1 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 10.0.0.1 dev net1
10.0.0.1 dev net1 scope link
20.0.0.0/24 via 10.0.0.1 dev net1
169.254.1.1 dev eth0 scope link

[root@vrouter2-1-pod1 /]# ip -6 route
abcd::a00:1 dev net1 metric 1024 pref medium
abcd::a00:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
abcd::1400:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev net1 proto kernel metric 256 pref medium
```
```
[root@vrouter2-2-pod2 /]# ip a
--- snip ---
284: net1@if285: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:17:d2:43 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.3/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe17:d243/64 scope link
       valid_lft forever preferred_lft forever

[root@vrouter2-2-pod2 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.1 dev net1 scope link
169.254.1.1 dev eth0 scope link

[root@vrouter2-2-pod2 /]# ip -6 route
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
set groups cni routing-instances vrouter2-2 instance-type virtual-router
set groups cni routing-instances vrouter2-2 routing-options rib vrouter2-2.inet6.0 static route abcd::1400:3/128 qualified-next-hop abcd::1400:3 interface jvknet1-2f190c2
set groups cni routing-instances vrouter2-2 routing-options static route 20.0.0.3/32 qualified-next-hop 20.0.0.3 interface jvknet1-2f190c2
set groups cni routing-instances vrouter2-2 interface jvknet1-2f190c2
set groups cni routing-instances vrouter2-1 instance-type virtual-router
set groups cni routing-instances vrouter2-1 routing-options rib vrouter2-1.inet6.0 static route abcd::a00:2/128 qualified-next-hop abcd::a00:2 interface jvknet1-62538eb
set groups cni routing-instances vrouter2-1 routing-options static route 10.0.0.2/32 qualified-next-hop 10.0.0.2 interface jvknet1-62538eb
set groups cni routing-instances vrouter2-1 interface jvknet1-62538eb
```

cRPD RouteLeak設定追加
```
root@jcnr1> show configuration | display set
set policy-options policy-statement import-from-vrouter2-1 term 1 from instance vrouter2-1
set policy-options policy-statement import-from-vrouter2-1 term 1 then accept
set policy-options policy-statement import-from-vrouter2-2 term 1 from instance vrouter2-2
set policy-options policy-statement import-from-vrouter2-2 term 1 then accept
set routing-instances vrouter2-1 routing-options instance-import import-from-vrouter2-2
set routing-instances vrouter2-2 routing-options instance-import import-from-vrouter2-1
```


cRPD Route確認
```
root@jcnr1> show route table vrouter2-1.inet.0

vrouter2-1.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 00:26:24
                    >  via jvknet1-62538eb
10.0.0.2/32        *[Local/0] 00:26:24
                       Local via jvknet1-62538eb
20.0.0.0/24        *[Direct/0] 00:19:16
                    >  via jvknet1-2f190c2
20.0.0.3/32        *[Local/0] 00:19:16
                       Local via jvknet1-2f190c2

root@jcnr1> show route table vrouter2-2.inet.0

vrouter2-2.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/24        *[Direct/0] 00:19:19
                    >  via jvknet1-62538eb
10.0.0.2/32        *[Local/0] 00:19:19
                       Local via jvknet1-62538eb
20.0.0.0/24        *[Direct/0] 00:19:19
                    >  via jvknet1-2f190c2
20.0.0.3/32        *[Local/0] 00:19:19
                       Local via jvknet1-2f190c2
```

vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-dpjmx -n contrail -- bash
bash-5.1# vif --list
--- snip ---
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:328 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:328  bytes:28268 errors:0
            TX packets:59325  bytes:8405974 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9000
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:L3L2Vof QOS:0 Ref:13
            RX device packets:85822  bytes:6857481 errors:0
            RX port   packets:85820 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:85820  bytes:6857313 errors:0
            TX packets:56492  bytes:4694549 errors:0
            Drops:17
            TX queue  packets:56312 errors:0
            TX port   packets:56492 errors:0
            TX device packets:56503  bytes:4695685 errors:0

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
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:15 TxXVif:1
            RX device packets:55289  bytes:4603414 errors:0
            RX queue  packets:55289 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:55289  bytes:4603414 errors:0
            TX packets:85393  bytes:6825338 errors:0
            Drops:0
            TX queue  packets:85393 errors:0
            TX device packets:85393  bytes:6825338 errors:0

vif0/6      Ethernet: jvknet1-62538eb NH: 16 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:10.0.0.2
            IP6addr:abcd::a00:2
            DDP: OFF SwLB: ON
            Vrf:2 Mcast Vrf:2 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:69 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:69  bytes:6731 errors:0
            TX packets:3  bytes:238 errors:0
            Drops:65
            TX queue  packets:2 errors:0
            TX port   packets:3 errors:0

vif0/7      Ethernet: jvknet1-2f190c2 NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:20.0.0.3
            IP6addr:abcd::1400:3
            DDP: OFF SwLB: ON
            Vrf:3 Mcast Vrf:3 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:50 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:50  bytes:6140 errors:0
            TX packets:7  bytes:582 errors:0
            Drops:42
            TX queue  packets:2 errors:0
            TX port   packets:7 errors:0
```

POD間疎通確認
```
[root@vrouter2-1-pod1 /]# ping 20.0.0.3
PING 20.0.0.3 (20.0.0.3) 56(84) bytes of data.
64 bytes from 20.0.0.3: icmp_seq=1 ttl=63 time=0.572 ms
64 bytes from 20.0.0.3: icmp_seq=2 ttl=63 time=0.111 ms
64 bytes from 20.0.0.3: icmp_seq=3 ttl=63 time=0.107 ms
```
