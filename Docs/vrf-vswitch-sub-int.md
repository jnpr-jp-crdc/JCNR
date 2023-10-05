動いてない。要確認

# VRF - vSwitch : Pod Vlan Sub Interface
- VRF Instance Type : Virtual Switch
- JCNR WorkerNode間のL2接続
- POD / JCNR vRouter間はVlan Tag接続　(1 vSwitch 1 Vlan Interface)
- POD VLAN InterfaceへのIP Assign　可能
- VLANはJCNRデプロイ時にL2 Interfaceに指定したVLANしか使用できません

## VRF vSwitch - Pod VLAN Sub Interface
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vswitch3.png" width=800>

### Network Attachment Definition　作成
[VRF vSwitch BD100 NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch3-bd100-nad-jcnr1.yaml)

[VRF vSwitch BD200 NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch3-bd200-nad-jcnr1.yaml)

[VRF vSwitch BD100 NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch3-bd100-nad-jcnr2.yaml)

[VRF vSwitch BD200 NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch3-bd200-nad-jcnr2.yaml)

#### NAD Option
- InstanceName : "vswitch"のまま。変更不可
- InstanceType : "virtual-switch"を使用
- bridgeDomain : BridgeDomain名
- bridgeVlanId : VlanID
- parentInterface : PODのParent Interface名
- interface : PodのSub Interface名
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch3-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch3-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
```
[root@jcnr1 yaml]# kubectl describe pod vswitch3-pod1
Name:         vswitch3-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Thu, 05 Oct 2023 01:12:09 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 89a595de2cc83b61bf096696a199790f0b780a9ea5029fa4883d12c9ba3588f6
              cni.projectcalico.org/podIP: 172.30.79.45/32
              cni.projectcalico.org/podIPs: 172.30.79.45/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.45"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch3-bd100",
                    "interface": "net1.100",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:25:9E:CF",
                    "dns": {}
                },{
                    "name": "default/vswitch3-bd200",
                    "interface": "net1.200",
                    "ips": [
                        "20.0.0.2",
                        "abcd::1400:2"
                    ],
                    "mac": "02:00:00:0A:13:E3",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch3-bd100",
                    "interface": "net1.100",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  },
                  {
                    "name": "vswitch3-bd200",
                    "interface": "net1.200",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```
```
[root@jcnr2 yaml]# kubectl describe pod vswitch3-pod2
Name:         vswitch3-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Thu, 05 Oct 2023 01:35:41 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: d7ba1ff1b1aa442957b443156c9b048cd59206fa5274b6d5619f07b0457d0973
              cni.projectcalico.org/podIP: 172.30.147.243/32
              cni.projectcalico.org/podIPs: 172.30.147.243/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.243"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch3-bd100",
                    "interface": "net1.100",
                    "ips": [
                        "10.0.0.3",
                        "abcd::a00:3"
                    ],
                    "mac": "02:00:00:42:04:8C",
                    "dns": {}
                },{
                    "name": "default/vswitch3-bd200",
                    "interface": "net1.200",
                    "ips": [
                        "20.0.0.3",
                        "abcd::1400:3"
                    ],
                    "mac": "02:00:00:5D:35:EF",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch3-bd100",
                    "interface": "net1.100",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  },
                  {
                    "name": "vswitch3-bd200",
                    "interface": "net1.200",
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
[root@vswitch3-pod1 /]# ip a
--- snip ---
265: net1@if266: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:25:9e:cf brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ff:fe25:9ecf/64 scope link
       valid_lft forever preferred_lft forever
10: net1.100@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:25:9e:cf brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1.100
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe25:9ecf/64 scope link
       valid_lft forever preferred_lft forever
11: net1.200@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:25:9e:cf brd ff:ff:ff:ff:ff:ff
    inet 20.0.0.2/24 brd 20.0.0.255 scope global net1.200
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe25:9ecf/64 scope link
       valid_lft forever preferred_lft forever
```
```
[root@vswitch3-pod2 /]# ip a
115: net1@if116: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:42:04:8c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ff:fe42:48c/64 scope link
       valid_lft forever preferred_lft forever
10: net1.100@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:42:04:8c brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.3/24 brd 10.0.0.255 scope global net1.100
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe42:48c/64 scope link
       valid_lft forever preferred_lft forever
11: net1.200@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:42:04:8c brd ff:ff:ff:ff:ff:ff
    inet 20.0.0.3/24 brd 20.0.0.255 scope global net1.200
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe42:48c/64 scope link
       valid_lft forever preferred_lft forever
```


cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni interfaces jvknet1-a5 unit 100 vlan-id 100
set groups cni interfaces jvknet1-a5 unit 200 vlan-id 200
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd100 vlan-id 100
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-a5.100
set groups cni routing-instances vswitch bridge-domains bd200 vlan-id 200
set groups cni routing-instances vswitch bridge-domains bd200 interface jvknet1-a5.200
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni interfaces jvknet1-5c unit 100 vlan-id 100
set groups cni interfaces jvknet1-5c unit 200 vlan-id 200
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd100 vlan-id 100
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-5c.100
set groups cni routing-instances vswitch bridge-domains bd200 vlan-id 200
set groups cni routing-instances vswitch bridge-domains bd200 interface jvknet1-5c.200
```

cRPD L2 Default設定
- JCNRデプロイ時に指定したInterface VLAN設定が以下の設定の通り反映されている
```
set interfaces ens5 native-vlan-id 100
set interfaces ens5 unit 0 family bridge interface-mode trunk
set interfaces ens5 unit 0 family bridge vlan-id-list 100
set interfaces ens5 unit 0 family bridge vlan-id-list 200
set interfaces ens5 unit 0 family bridge vlan-id-list 300
set interfaces ens5 unit 0 family bridge vlan-id-list 700-705
set interfaces ens5 unit 0 family bridge ce-facing
set routing-instances vswitch instance-type virtual-switch
set routing-instances vswitch bridge-domains bd100 vlan-id 100
set routing-instances vswitch bridge-domains bd200 vlan-id 200
set routing-instances vswitch bridge-domains bd300 vlan-id 300
set routing-instances vswitch bridge-domains bd700 vlan-id 700
set routing-instances vswitch bridge-domains bd701 vlan-id 701
set routing-instances vswitch bridge-domains bd702 vlan-id 702
set routing-instances vswitch bridge-domains bd703 vlan-id 703
set routing-instances vswitch bridge-domains bd704 vlan-id 704
set routing-instances vswitch bridge-domains bd705 vlan-id 705
set routing-instances vswitch interface ens5
```

cRPD Bridge Domain Mac Table 確認
```
エントリなし
```

vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-flnlk -n contrail -- bash
bash-5.1# vif --list
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:272 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:272  bytes:23392 errors:0
            TX packets:59171  bytes:8386458 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:11
            RX device packets:82162  bytes:6583473 errors:0
            RX port   packets:82160 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:82160  bytes:6583305 errors:0
            TX packets:54720  bytes:4544095 errors:0
            Drops:0
            TX queue  packets:54720 errors:0
            TX port   packets:54720 errors:0
            TX device packets:54731  bytes:4545231 errors:0

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

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:13 TxXVif:1
            RX device packets:54245  bytes:4512896 errors:0
            RX queue  packets:54245 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:54245  bytes:4512896 errors:0
            TX packets:82160  bytes:6583309 errors:0
            Drops:0
            TX queue  packets:82160 errors:0
            TX device packets:82160  bytes:6583309 errors:0

vif0/6      Ethernet: jvknet1-a5 NH: 16 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:65535 Mcast Vrf:65535 Flags:L3DVofProxyEr QOS:-1 Ref:15
            RX port   packets:83 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:8  bytes:560 errors:0
            TX packets:5  bytes:230 errors:0
            Drops:8
            TX port   packets:5 errors:0

vif0/7      Virtual: jvknet1-a5.100 Vlan(o/i)(,S): 100/100 MTU: 1514
            Parent:vif0/6
            Type:Virtual(Vlan) HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:65535 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:4
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:27  bytes:2418 errors:0
            TX packets:2  bytes:84 errors:0
            Drops:25

vif0/8      Virtual: jvknet1-a5.200 Vlan(o/i)(,S): 200/200 NH: 19 MTU: 1514
            Parent:vif0/6
            Type:Virtual(Vlan) HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:65535 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:4
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:48  bytes:4504 errors:0
            TX packets:3  bytes:126 errors:0
            Drops:45
```
```
[root@jcnr2 yaml]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:1562 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:1562  bytes:135108 errors:0
            TX packets:1794  bytes:172720 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:13
            RX device packets:113798  bytes:9438207 errors:0
            RX port   packets:113786 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:113786  bytes:9434420 errors:0
            TX packets:177524  bytes:15305821 errors:0
            Drops:111
            TX queue  packets:176504 errors:0
            TX port   packets:177524 errors:0
            TX device packets:177551  bytes:15308151 errors:0

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

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:15 TxXVif:1
            RX device packets:167961  bytes:14480227 errors:0
            RX queue  packets:167961 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:167961  bytes:14480227 errors:0
            TX packets:112196  bytes:9331076 errors:0
            Drops:0
            TX queue  packets:112196 errors:0
            TX device packets:112196  bytes:9331076 errors:0

vif0/6      Ethernet: jvknet1-5c NH: 14 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:65535 Mcast Vrf:65535 Flags:L3DVofProxyEr QOS:-1 Ref:15
            RX port   packets:27 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:16  bytes:1468 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:16

vif0/7      Virtual: jvknet1-5c.100 Vlan(o/i)(,S): 100/100 NH: 21 MTU: 1514
            Parent:vif0/6
            Type:Virtual(Vlan) HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:65535 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:4
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:5  bytes:370 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:5

vif0/8      Virtual: jvknet1-5c.200 Vlan(o/i)(,S): 200/200 NH: 24 MTU: 1514
            Parent:vif0/6
            Type:Virtual(Vlan) HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:65535 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:4
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:6  bytes:444 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:6
```

POD間疎通確認
```
[root@vswitch2-pod1 /]# ping 71.0.0.2
PING 71.0.0.2 (71.0.0.2) 56(84) bytes of data.
64 bytes from 71.0.0.2: icmp_seq=1 ttl=64 time=0.544 ms
64 bytes from 71.0.0.2: icmp_seq=2 ttl=64 time=0.240 ms
```
