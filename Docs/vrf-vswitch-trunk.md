# VRF - vSwitch : Pod Vlan Trunk
- VRF Instance Type : Virtual Switch
- JCNR WorkerNode間のL2接続
- POD / JCNR vRouter間はVlan Trunk接続
- PODへのIP Assign不可

## VRF vSwitch - Pod VLAN Trunk
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vswitch2.png" width=600>

### Network Attachment Definition　作成
[VRF vSwitch NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-nad-jcnr1.yaml)

[VRF vSwitch NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-nad-jcnr2.yaml)




#### NAD Option
- InstanceName : "vswitch"のまま。変更不可
- InstanceType : VRF Liteは"virtual-switch"を使用
- vlanIdList: VLAN ID
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
```
[root@jcnr1 yaml]# kubectl describe pod vswitch2-pod1
Name:         vswitch2-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Wed, 04 Oct 2023 22:27:52 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 6ae2a57c22bbd638148d1d29c53adae784ce4bda055d5e5cd19cded3e8c3b9b3
              cni.projectcalico.org/podIP: 172.30.79.44/32
              cni.projectcalico.org/podIPs: 172.30.79.44/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.44"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch2",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:25:B8:DB",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch2",
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
[root@jcnr2 yaml]# kubectl describe pod vswitch2-pod2
Name:         vswitch2-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Wed, 04 Oct 2023 22:28:21 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: d20b253fcd1abea5dca82dfc121670545c0e72f458e3e2d406e1c07238bbc8cb
              cni.projectcalico.org/podIP: 172.30.147.242/32
              cni.projectcalico.org/podIPs: 172.30.147.242/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.242"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch2",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.3",
                        "abcd::a00:3"
                    ],
                    "mac": "02:00:00:D5:93:CA",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch2",
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
[root@vswitch2-pod1 /]# ip a
--- snip ---
263: net1@if264: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:25:b8:db brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe25:b8db/64 scope link
       valid_lft forever preferred_lft forever

[root@vswitch2-pod1 /]# ip link add link net1 name net1.100 type vlan id 100
[root@vswitch2-pod1 /]# ip addr add dev net1.100 100.0.0.2/24
[root@vswitch2-pod1 /]# ip link set net1.100 up
[root@vswitch2-pod1 /]# ip link add link net1 name net1.200 type vlan id 200
[root@vswitch2-pod1 /]# ip addr add dev net1.200 200.0.0.2/24
[root@vswitch2-pod1 /]# ip link set net1.200 up
[root@vswitch2-pod1 /]# ip a
--- snip ---
10: net1.100@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:00:00:25:b8:db brd ff:ff:ff:ff:ff:ff
    inet 100.0.0.2/24 scope global net1.100
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe25:b8db/64 scope link
       valid_lft forever preferred_lft forever
11: net1.200@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:00:00:25:b8:db brd ff:ff:ff:ff:ff:ff
    inet 200.0.0.2/24 scope global net1.200
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe25:b8db/64 scope link
       valid_lft forever preferred_lft forever

```
```
[root@vswitch2-pod2 /]# ip a
--- snip ---
113: net1@if114: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:d5:93:ca brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.3/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fed5:93ca/64 scope link
       valid_lft forever preferred_lft forever

[root@vswitch2-pod2 /]# ip link add link net1 name net1.100 type vlan id 100
[root@vswitch2-pod2 /]# ip addr add dev net1.100 100.0.0.3/24
[root@vswitch2-pod2 /]# ip link set net1.100 up
[root@vswitch2-pod2 /]# ip link add link net1 name net1.200 type vlan id 200
[root@vswitch2-pod2 /]# ip addr add dev net1.200 200.0.0.3/24
[root@vswitch2-pod2 /]# ip link set net1.200 up
[root@vswitch2-pod2 /]# ip a
10: net1.100@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:00:00:d5:93:ca brd ff:ff:ff:ff:ff:ff
    inet 100.0.0.3/24 scope global net1.100
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fed5:93ca/64 scope link
       valid_lft forever preferred_lft forever
11: net1.200@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:00:00:d5:93:ca brd ff:ff:ff:ff:ff:ff
    inet 200.0.0.3/24 scope global net1.200
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fed5:93ca/64 scope link
       valid_lft forever preferred_lft forever
```


cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni interfaces jvknet1-f2f88f7 unit 0 family bridge interface-mode trunk
set groups cni interfaces jvknet1-f2f88f7 unit 0 family bridge vlan-id-list 100
set groups cni interfaces jvknet1-f2f88f7 unit 0 family bridge vlan-id-list 200
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch interface jvknet1-f2f88f7
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni interfaces jvknet1-e2cd3e3 unit 0 family bridge interface-mode trunk
set groups cni interfaces jvknet1-e2cd3e3 unit 0 family bridge vlan-id-list 100
set groups cni interfaces jvknet1-e2cd3e3 unit 0 family bridge vlan-id-list 200
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch interface jvknet1-e2cd3e3
```

cRPD L2 Default設定
- JCNRデプロイ時に指定したInterface VLAN設定が以下の設定の通り反映されている
- 下記以外のVLANIDを使用する場合は、都度下記編集
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
root@jcnr1> show bridge mac-table

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 100
MAC                  MAC                Logical
address              flags              interface

02:00:00:25:b8:db      D                 jvknet1-f2f88f7
02:00:00:d5:93:ca      D                 ens5

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 200
MAC                  MAC                Logical
address              flags              interface

02:00:00:25:b8:db      D                 jvknet1-f2f88f7
02:00:00:d5:93:ca      D                 ens5
```
```
root@jcnr2> show bridge mac-table

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 100
MAC                  MAC                Logical
address              flags              interface

02:00:00:25:b8:db      D                 ens5
02:00:00:d5:93:ca      D                 jvknet1-e2cd3e3

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 200
MAC                  MAC                Logical
address              flags              interface

02:00:00:25:b8:db      D                 ens5
02:00:00:d5:93:ca      D                 jvknet1-e2cd3e3
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
            TX packets:59155  bytes:8384570 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:11
            RX device packets:76577  bytes:6170102 errors:0
            RX port   packets:76575 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:76575  bytes:6169934 errors:0
            TX packets:53123  bytes:4414160 errors:0
            Drops:0
            TX queue  packets:53123 errors:0
            TX port   packets:53123 errors:0
            TX device packets:53134  bytes:4415296 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:163  bytes:12830 errors:0
            RX port   packets:163 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:163  bytes:13198 errors:0
            TX packets:190  bytes:14184 errors:0
            Drops:49
            TX queue  packets:190 errors:0
            TX port   packets:190 errors:0
            TX device packets:190  bytes:14184 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:13 TxXVif:1
            RX device packets:52648  bytes:4382961 errors:0
            RX queue  packets:52648 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:52648  bytes:4382961 errors:0
            TX packets:76575  bytes:6169938 errors:0
            Drops:0
            TX queue  packets:76575 errors:0
            TX device packets:76575  bytes:6169938 errors:0

vif0/6      Ethernet: jvknet1-f2f88f7 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:71 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Trunk  Vlan: 100 200
            RX packets:52  bytes:7231 errors:25
            TX packets:47  bytes:3682 errors:0
            Drops:52
            TX queue  packets:47 errors:0
            TX port   packets:47 errors:0
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
            TX packets:1778  bytes:170832 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:13
            RX device packets:112198  bytes:9308061 errors:0
            RX port   packets:112186 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:112186  bytes:9304274 errors:0
            TX packets:171928  bytes:14891673 errors:0
            Drops:111
            TX queue  packets:170908 errors:0
            TX port   packets:171928 errors:0
            TX device packets:171955  bytes:14894003 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:ac:04:91
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:236  bytes:18152 errors:0
            RX port   packets:236 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:209  bytes:17258 errors:27
            TX packets:257  bytes:27105 errors:0
            Drops:57
            TX queue  packets:257 errors:0
            TX port   packets:257 errors:0
            TX device packets:257  bytes:27105 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:15 TxXVif:1
            RX device packets:162365  bytes:14066079 errors:0
            RX queue  packets:162365 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:162365  bytes:14066079 errors:0
            TX packets:110596  bytes:9200930 errors:0
            Drops:0
            TX queue  packets:110596 errors:0
            TX device packets:110596  bytes:9200930 errors:0

vif0/6      Ethernet: jvknet1-e2cd3e3 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:75 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Trunk  Vlan: 100 200
            RX packets:60  bytes:4735 errors:28
            TX packets:46  bytes:3588 errors:0
            Drops:45
            TX queue  packets:46 errors:0
            TX port   packets:46 errors:0
```

POD間疎通確認
```
[root@vswitch2-pod1 /]# ping 100.0.0.3
PING 100.0.0.3 (100.0.0.3) 56(84) bytes of data.
64 bytes from 100.0.0.3: icmp_seq=1 ttl=64 time=0.412 ms
64 bytes from 100.0.0.3: icmp_seq=2 ttl=64 time=0.243 ms

[root@vswitch2-pod1 /]# ping 200.0.0.3
PING 200.0.0.3 (200.0.0.3) 56(84) bytes of data.
64 bytes from 200.0.0.3: icmp_seq=1 ttl=64 time=0.289 ms
64 bytes from 200.0.0.3: icmp_seq=2 ttl=64 time=0.247 ms
```
