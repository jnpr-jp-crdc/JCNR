# VRF - vSwitch : Pod Vlan Access
- VRF Instance Type : Virtual Switch
- JCNR WorkerNode間のL2接続
- POD / JCNR vRouter間はVlan Access接続
- VLANはJCNRデプロイ時にL2 Interfaceに指定したVLANしか使用できません

## VRF vSwitch - Pod VLAN Access 
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vswitch1.png" width=600>

### Network Attachment Definition　作成
[VRF vSwitch BD100 NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-nad-bd100-jcnr1.yaml)

[VRF vSwitch BD200 NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-nad-bd200-jcnr1.yaml)

[VRF vSwitch BD100 NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-nad-bd100-jcnr2.yaml)

[VRF vSwitch BD200 NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-nad-bd200-jcnr2.yaml)

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
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
```
[root@jcnr1]# kubectl describe pod vswitch1-pod1
Name:         vswitch1-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Wed, 04 Oct 2023 21:41:43 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: e2582a8ea1f2414d96b00f81ecb1a1df2581b3d88d42d59406cf65cedd102f70
              cni.projectcalico.org/podIP: 172.30.79.41/32
              cni.projectcalico.org/podIPs: 172.30.79.41/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.41"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch1-bd100",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:2C:60:91",
                    "dns": {}
                },{
                    "name": "default/vswitch1-bd200",
                    "interface": "net2",
                    "ips": [
                        "20.0.0.2",
                        "abcd::1400:2"
                    ],
                    "mac": "02:00:00:9D:E9:7C",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch1-bd100",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  },
                  {
                    "name": "vswitch1-bd200",
                    "interface":"net2",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```
```
[root@jcnr2]# kubectl describe pod vswitch1-pod2
Name:         vswitch1-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Wed, 04 Oct 2023 21:47:14 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: af4501d199f301f13d19ee493165efd529d0355a1ae4fa3835a40ffdf08741ab
              cni.projectcalico.org/podIP: 172.30.147.239/32
              cni.projectcalico.org/podIPs: 172.30.147.239/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.239"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch1-bd100",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.3",
                        "abcd::a00:3"
                    ],
                    "mac": "02:00:00:88:C7:FC",
                    "dns": {}
                },{
                    "name": "default/vswitch1-bd200",
                    "interface": "net2",
                    "ips": [
                        "20.0.0.3",
                        "abcd::1400:3"
                    ],
                    "mac": "02:00:00:5C:C0:A7",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch1-bd100",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  },
                  {
                    "name": "vswitch1-bd200",
                    "interface":"net2",
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
[root@vswitch1-pod1 /]# ip a
--- snip ---
255: net1@if256: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:2c:60:91 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe2c:6091/64 scope link
       valid_lft forever preferred_lft forever
257: net2@if258: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:9d:e9:7c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.2/24 brd 20.0.0.255 scope global net2
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe9d:e97c/64 scope link
       valid_lft forever preferred_lft forever
```
```
[root@vswitch1-pod2 /]# ip a
--- snip ---
103: net1@if104: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:88:c7:fc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.3/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe88:c7fc/64 scope link
       valid_lft forever preferred_lft forever
105: net2@if106: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:5c:c0:a7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.3/24 brd 20.0.0.255 scope global net2
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe5c:c0a7/64 scope link
       valid_lft forever preferred_lft forever
```


cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd100 vlan-id 100
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-d7b1c60
set groups cni routing-instances vswitch bridge-domains bd200 vlan-id 200
set groups cni routing-instances vswitch bridge-domains bd200 interface jvknet2-d7b1c60
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd100 vlan-id 100
set groups cni routing-instances vswitch bridge-domains bd100 interface jvknet1-766dede
set groups cni routing-instances vswitch bridge-domains bd200 vlan-id 200
set groups cni routing-instances vswitch bridge-domains bd200 interface jvknet2-766dede
```

cRPD L2 Default設定
- JCNRデプロイ時に指定したInterface VLAN設定が以下の設定の通り反映されている
```
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

02:00:00:2c:60:91      D                 jvknet1-d7b1c60
02:00:00:88:c7:fc      D                 ens5

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 200
MAC                  MAC                Logical
address              flags              interface

00:00:5e:00:01:00      D                 jvknet2-d7b1c60
02:00:00:5c:c0:a7      D                 ens5
02:00:00:9d:e9:7c      D                 jvknet2-d7b1c60
```
```
root@jcnr2> show bridge mac-table

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 100
MAC                  MAC                Logical
address              flags              interface

00:00:5e:00:01:00      D                 ens5
02:00:00:2c:60:91      D                 ens5
02:00:00:88:c7:fc      D                 jvknet1-766dede

MAC flags         (S - Static MAC, D - Dynamic MAC)
Routing Instance : default-domain:contrail:ip-fabric:default
Bridging domain VLAN id : 200
MAC                  MAC                Logical
address              flags              interface

00:00:5e:00:01:00      D                 ens5
02:00:00:5c:c0:a7      D                 jvknet2-766dede
02:00:00:9d:e9:7c      D                 ens5
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
            TX packets:59120  bytes:8380164 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:11
            RX device packets:75246  bytes:6070172 errors:0
            RX port   packets:75244 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:75244  bytes:6070004 errors:0
            TX packets:52732  bytes:4381342 errors:0
            Drops:0
            TX queue  packets:52732 errors:0
            TX port   packets:52732 errors:0
            TX device packets:52743  bytes:4382478 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:60:b1:d8
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:57  bytes:4434 errors:0
            RX port   packets:57 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:57  bytes:4566 errors:0
            TX packets:63  bytes:4930 errors:0
            Drops:13
            TX queue  packets:63 errors:0
            TX port   packets:63 errors:0
            TX device packets:63  bytes:4930 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:43:f9:a1 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:13 TxXVif:1
            RX device packets:52257  bytes:4350143 errors:0
            RX queue  packets:52257 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:52257  bytes:4350143 errors:0
            TX packets:75244  bytes:6070008 errors:0
            Drops:0
            TX queue  packets:75244 errors:0
            TX device packets:75244  bytes:6070008 errors:0

vif0/6      Ethernet: jvknet1-d7b1c60 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:34 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 100  OVlan Id: 100
            RX packets:40  bytes:4492 errors:0
            TX packets:33  bytes:2538 errors:0
            Drops:10
            TX queue  packets:33 errors:0
            TX port   packets:33 errors:0

vif0/7      Ethernet: jvknet2-d7b1c60 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:29 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 200  OVlan Id: 200
            RX packets:34  bytes:2886 errors:0
            TX packets:24  bytes:1800 errors:0
            Drops:10
            TX queue  packets:24 errors:0
            TX port   packets:24 errors:0
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
            TX packets:1741  bytes:165382 errors:0
            Drops:0

vif0/1      PCI: 0000:00:04.0 NH: 6 MTU: 9014
            Type:Physical HWaddr:52:54:00:cf:74:89 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:0 Flags:L3L2Vof QOS:0 Ref:13
            RX device packets:111818  bytes:9276090 errors:0
            RX port   packets:111806 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:04.0  Status: UP  Driver: net_virtio
            RX packets:111806  bytes:9272303 errors:0
            TX packets:170636  bytes:14794825 errors:0
            Drops:111
            TX queue  packets:169616 errors:0
            TX port   packets:170636 errors:0
            TX device packets:170663  bytes:14797155 errors:0

vif0/2      PCI: 0000:00:05.0 MTU: 9000
            Type:Physical HWaddr:52:54:00:ac:04:91
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:9
            RX device packets:109  bytes:8898 errors:0
            RX port   packets:109 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Fabric Interface: 0000:00:05.0  Status: UP  Driver: net_virtio
            Vlan Mode: Trunk  Vlan: 100 200 300 700-705
            Native vlan id: 100
            RX packets:109  bytes:9070 errors:0
            TX packets:148  bytes:17867 errors:0
            Drops:15
            TX queue  packets:148 errors:0
            TX port   packets:148 errors:0
            TX device packets:148  bytes:17867 errors:0

vif0/3      PMD: ens4 NH: 10 MTU: 9014
            Type:Host HWaddr:52:54:00:cf:74:89 IPaddr:0.0.0.0
            DDP: OFF SwLB: ON
            Vrf:0 Mcast Vrf:65535 Flags:L3DProxyEr QOS:-1 Ref:15 TxXVif:1
            RX device packets:161073  bytes:13969231 errors:0
            RX queue  packets:161073 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:161073  bytes:13969231 errors:0
            TX packets:110216  bytes:9168959 errors:0
            Drops:0
            TX queue  packets:110216 errors:0
            TX device packets:110216  bytes:9168959 errors:0

vif0/6      Ethernet: jvknet1-766dede MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:70 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 100  OVlan Id: 100
            RX packets:96  bytes:15758 errors:0
            TX packets:12  bytes:840 errors:0
            Drops:69
            TX queue  packets:12 errors:0
            TX port   packets:12 errors:0

vif0/7      Ethernet: jvknet2-766dede MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:30 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 200  OVlan Id: 200
            RX packets:43  bytes:6140 errors:0
            TX packets:8  bytes:560 errors:0
            Drops:27
            TX queue  packets:8 errors:0
            TX port   packets:8 errors:0
```

POD間疎通確認
```
[root@vswitch1-pod1 /]# ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.269 ms
64 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.230 ms
64 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.240 ms

[root@vswitch1-pod1 /]# ping 20.0.0.3
PING 20.0.0.3 (20.0.0.3) 56(84) bytes of data.
64 bytes from 20.0.0.3: icmp_seq=1 ttl=64 time=0.260 ms
64 bytes from 20.0.0.3: icmp_seq=2 ttl=64 time=0.268 ms
64 bytes from 20.0.0.3: icmp_seq=3 ttl=64 time=0.239 ms
```
