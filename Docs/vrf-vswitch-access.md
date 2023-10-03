# VRF - vswitch
- VRF Instance Type : Virtual Switch
- JCNR WorkerNode間のL2接続

## VRF vSwitch - Pod VLAN Access 
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vswitch1.png" width=400>

### Network Attachment Definition　作成
[VRF vSwitch NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-nad-jcnr1.yaml)
[VRF vSwitch NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch1-nad-jcnr2.yaml)




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
Start Time:   Tue, 03 Oct 2023 03:22:20 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 7d3b02e8292c15618f078abca50ebd6513eb8a66c257ca13430e0737d0d1cbbe
              cni.projectcalico.org/podIP: 172.30.79.25/32
              cni.projectcalico.org/podIPs: 172.30.79.25/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.25"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:E3:15:B7",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch1",
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
[root@jcnr2]# kubectl describe pod vswitch1-pod2
Name:         vswitch1-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Tue, 03 Oct 2023 03:26:56 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 0b23eba122c7ccc502761dce6240734822d03242ce990384bbd325b2d4c26494
              cni.projectcalico.org/podIP: 172.30.147.230/32
              cni.projectcalico.org/podIPs: 172.30.147.230/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.230"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.3",
                        "abcd::a00:3"
                    ],
                    "mac": "02:00:00:F0:DE:76",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch1",
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
[root@vswitch1-pod1 /]# ip a
--- snip ---
185: net1@if186: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:e3:15:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fee3:15b7/64 scope link
       valid_lft forever preferred_lft forever
```
```
[root@vswitch1-pod2 /]# ip a
--- snip ---
60: net1@if61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:f0:de:76 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.3/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fef0:de76/64 scope link
       valid_lft forever preferred_lft forever
```


cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd300 vlan-id 300
set groups cni routing-instances vswitch bridge-domains bd300 interface jvknet1-a47ad95
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch bridge-domains bd300 vlan-id 300
set groups cni routing-instances vswitch bridge-domains bd300 interface jvknet1-26814fb
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

vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-flnlk -n contrail -- bash
bash-5.1# vif --list
vif0/6      Ethernet: jvknet1-a47ad95 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:24 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Access  Vlan Id: 300  OVlan Id: 300
            RX packets:26  bytes:2116 errors:1
            TX packets:38  bytes:3040 errors:0
            Drops:10
            TX queue  packets:38 errors:0
            TX port   packets:38 errors:0
```
