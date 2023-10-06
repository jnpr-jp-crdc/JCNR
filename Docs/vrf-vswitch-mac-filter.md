# VRF - vSwitch : Mac Filter
- VRF Instance Type : Virtual Switch
- POD / JCNR vRouter間はVlan Access接続
- Bridge Domain内でMacによるFilterを実施

## VRF vSwitch - Pod VLAN Access 
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vswitch4.png" width=600>

### Network Attachment Definition　作成
[VRF vSwitch BD100 NAD Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch4-nad.yaml)

#### NAD Option
- InstanceName : "vswitch"のまま。変更不可
- InstanceType : VRF Liteは"virtual-switch"を使用
- bridgeDomain: BridgeDomain名。名称はbd + VLANIDとする
- bridgeVlanId: VLAN ID
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch4-pod1.yaml)

[Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch4-pod2.yaml)

[Pod3 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch4-pod3.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
```
[root@jcnr1]# kubectl describe pod vswitch4-pod1
Name:         vswitch4-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Fri, 06 Oct 2023 02:50:18 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: f92da7791538d40da9852f1eff544c3d8c00732eea15d9ff5d95ee16b6ae25ed
              cni.projectcalico.org/podIP: 172.30.79.56/32
              cni.projectcalico.org/podIPs: 172.30.79.56/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.56"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch4",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:14:53:72",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch4",
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
[root@jcnr1]# kubectl describe pod vswitch4-pod2
Name:         vswitch4-pod2
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Fri, 06 Oct 2023 02:50:40 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 718996ea5eff6b09d42fbf8a46a9d16bff808640753930ea7f487f238ef7357d
              cni.projectcalico.org/podIP: 172.30.79.58/32
              cni.projectcalico.org/podIPs: 172.30.79.58/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.58"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch4",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.3",
                        "abcd::a00:3"
                    ],
                    "mac": "02:00:00:80:B7:98",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch4",
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
[root@jcnr1]# kubectl describe pod vswitch4-pod3
Name:         vswitch4-pod3
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Fri, 06 Oct 2023 02:50:43 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: e41e6385fd26a6ecf0155811ba71323c12e646b6386fdb397bad63f75b86add8
              cni.projectcalico.org/podIP: 172.30.79.59/32
              cni.projectcalico.org/podIPs: 172.30.79.59/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.59"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch4",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.4",
                        "abcd::a00:4"
                    ],
                    "mac": "02:00:00:07:A4:4A",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch4",
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
[root@vswitch4-pod1 /]# ip a
--- snip ---
324: net1@if325: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:14:53:72 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe14:5372/64 scope link
       valid_lft forever preferred_lft forever
```
```
[root@vswitch4-pod2 /]# ip a
--- snip ---
327: net1@if328: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:80:b7:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.3/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe80:b798/64 scope link
       valid_lft forever preferred_lft forever
```
```
[root@vswitch4-pod3 /]# ip a
--- snip ---
330: net1@if331: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:07:a4:4a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.4/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:4/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe07:a44a/64 scope link
       valid_lft forever preferred_lft forever
```

cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd100 vlan-id 100
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-5f0a787
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-897ee05
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-e1c1e9f
```

cRPD L2 Default設定
- JCNRデプロイ時に指定したInterface VLAN設定が以下の設定の通り反映されている
- 下記以外のBridgeDomain/VLANIDを使用する場合は、下記を編集
```
set interfaces ens5 native-vlan-id 100
set interfaces ens5 unit 0 family bridge interface-mode trunk
set interfaces ens5 unit 0 family bridge vlan-id-list 100
set interfaces ens5 unit 0 family bridge vlan-id-list 200
set interfaces ens5 unit 0 family bridge vlan-id-list 300
set interfaces ens5 unit 0 family bridge vlan-id-list 400
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

cRPD Mac Table 確認
```
root@jcnr1> show bridge mac-table

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 100
MAC                  MAC                Logical
address              flags              interface

00:00:5e:00:01:00      D                 jvknet1-5f0a787
02:00:00:07:a4:4a      D                 jvknet1-5f0a787
02:00:00:14:53:72      D                 jvknet1-897ee05
02:00:00:80:b7:98      D                 jvknet1-e1c1e9f
```

vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-flnlk -n contrail -- bash
bash-5.1# vif --list
vif0/0      Socket: unix MTU: 1514
            Type:Agent HWaddr:00:00:5e:00:01:00
            Vrf:65535 Flags:L2 QOS:-1 Ref:3
            RX port   packets:128 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:128  bytes:11348 errors:0
            TX packets:192  bytes:19753 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:192.168.0.1
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:14
            RX device packets:35590  bytes:3396407 errors:0
            RX port   packets:35590 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:35590  bytes:3396407 errors:0
            TX packets:14617  bytes:1265806 errors:0
            Drops:0
            TX queue  packets:14572 errors:0
            TX port   packets:14617 errors:0
            TX device packets:14638  bytes:1267721 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:14  bytes:1652 errors:0
            RX port   packets:7 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:14  bytes:1680 errors:7
            TX packets:88  bytes:6900 errors:0
            Drops:8
            TX queue  packets:62 errors:0
            TX port   packets:88 errors:0
            TX device packets:88  bytes:6900 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9000
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2 QOS:0 Ref:13 TxXVif:1
            RX device packets:14078  bytes:1220600 errors:0
            RX queue  packets:14078 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:14078  bytes:1220600 errors:0
            TX packets:23102  bytes:2125175 errors:0
            Drops:0
            TX queue  packets:23102 errors:0
            TX device packets:23102  bytes:2125175 errors:0

vif0/6      Ethernet: jvknet1-897ee05 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:56 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 100  OVlan Id: 100
            RX packets:76  bytes:16283 errors:0
            TX packets:80  bytes:6058 errors:0
            Drops:37
            TX queue  packets:80 errors:0
            TX port   packets:80 errors:0

vif0/7      Ethernet: jvknet1-e1c1e9f MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:62 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 100  OVlan Id: 100
            RX packets:62  bytes:4844 errors:0
            TX packets:67  bytes:4894 errors:0
            Drops:21
            TX queue  packets:67 errors:0
            TX port   packets:67 errors:0

vif0/8      Ethernet: jvknet1-5f0a787 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:57 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 100  OVlan Id: 100
            RX packets:62  bytes:5440 errors:0
            TX packets:60  bytes:4256 errors:0
            Drops:19
            TX queue  packets:60 errors:0
            TX port   packets:60 errors:0
```


POD間疎通確認
```
[root@vswitch4-pod1 /]# ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=0.081 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=0.089 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=64 time=0.074 ms

[root@vswitch4-pod2 /]# ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=0.121 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=0.093 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=64 time=0.085 ms
```

### Filter設定
```
set firewall:firewall family bridge filter filter1 term t1 from destination-mac-address 02:00:00:07:a4:4a
set firewall:firewall family bridge filter filter1 term t1 from source-mac-address 02:00:00:80:b7:98
set firewall:firewall family bridge filter filter1 term t1 then discard
set routing-instances vswitch bridge-domains bd100 forwarding-options filter input filter1
```

POD間疎通確認
```
[root@vswitch4-pod1 /]# ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
64 bytes from 10.0.0.4: icmp_seq=1 ttl=64 time=0.100 ms
64 bytes from 10.0.0.4: icmp_seq=2 ttl=64 time=0.098 ms
64 bytes from 10.0.0.4: icmp_seq=3 ttl=64 time=4.05 ms

[root@vswitch4-pod2 /]# ping 10.0.0.4
PING 10.0.0.4 (10.0.0.4) 56(84) bytes of data.
^C
--- 10.0.0.4 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2068ms
```

Filter Count確認
- FilterにMatchしDropしたPacket Countを確認
```
root@jcnr1> show firewall filter filter1

 Filter : filter1    vlan-id : 100
 Term                  Packet
  t1                   10
```
